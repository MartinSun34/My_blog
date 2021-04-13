## How does Postgres handle external parameters?

There are many advantages to using bind variables in SQL statements, especially in conditional statements. Using bind variables can save the trouble to parse and optimize the query in every execution. There are also disadvantages such as bind variable will make optimizer lost its prediction accuracy and cause performance decrease. In PostgreSQL are represented by $ sign preceded with a number, just like '$n'. Have you ever wondered how to pass and process these parameters in Postgres? In general, a query with bind variables will be transformed to a query tree with Param nodes. Then when this query get executed, it will fetch the input parameters stored in execution state. More details will be introduced below.



### how to handle placeholder in query

**In parser**
When the Postgres parser scans the dollar sign and followed by integers in the input query string, the Parser will treat it as a 'PARAM' token and make a ParamRef node based on the information and insert it into the parsetree. If the parameter is a composite type, you can use dot notation (such as $1.name) to access the attribute of the parameter.

```
| PARAM opt_indirection
	{
		ParamRef *p = makeNode(ParamRef);
		p->number = $1;
		p->location = @1;
		if ($2)
		{
			A_Indirection *n = makeNode(A_Indirection);
			n->arg = (Node *) p;
			n->indirection = check_indirection($2, yyscanner);
			$$ = (Node *) n;
		}
		else
			$$ = (Node *) p;
	}
```

**In analyzer**
In Postgres, there is a primitive node type called 'Param'. It represents various types of parameters in the query/plan tree. There are four kinds of Param nodes: 

- PARAM_EXTERN: The parameter value is supplied from outside the plan. 
- PARAM_EXEC: The parameter is an internal executor parameter, used for passing values into and out of sub-queries or from nestloop joins to their inner scans. 
- PARAM_SUBLINK: The parameter represents an output column of a SubLink node's sub-select. (This type of Param is converted to PARAM_EXEC during planning.)
- PARAM_MULTIEXPR: Like PARAM_SUBLINK, the parameter represents an output column of a SubLink node's sub-select, but here, the SubLink is always a MULTIEXPR SubLink. (This type of Param is also converted to PARAM_EXEC during planning.)

Except for PARAM_EXTERN, the other three represent internal parameters during query execution. This blog will mainly introduce external parameters. When the analyzer traverses the raw_parse_tree, when it encounters a ParamRef node, the analyzer will generate a Param node based on the information in the ParamRef node, and mark the Param node with kind ‘PARAM_EXTERN’. 

```c
typedef enum ParamKind
{
	PARAM_EXTERN,
	PARAM_EXEC,
	PARAM_SUBLINK,
	PARAM_MULTIEXPR
} ParamKind;

typedef struct Param
{
	Expr		xpr;
	ParamKind	paramkind;		/* kind of parameter. See above */
	int			paramid;		/* numeric ID for parameter */
	Oid			paramtype;		/* pg_type OID of parameter's datatype */
	int32		paramtypmod;	/* typmod value, if known */
	Oid			paramcollid;	/* OID of collation, or InvalidOid if none */
	int			location;		/* token location, or -1 if unknown */
} Param;
```

There are several functions can process this node depends on which module this sql statement comes from. 

- For example when a  query defined in a SQL function, analyzer will call sql_fn_param_ref() to handle ParamRef node. 
- For a query in plsql function,  plpgsql_param_ref() will do the parse. 
- For a prepared statement things will be a little tricky. When creating a prepared statement, you do not need to declare the type of the parameter, and this information will be inferred based on the context. Function variable_paramref_hook() will handle this situation build a Param node and mark param type as unknown type. Later variable_coerce_param_hook() will get the target type and assign the type id to corresponding Param node.

| ParamRef node in Query | Callback function to handle this case                      |
| ---------------------- | ---------------------------------------------------------- |
| In sql function        | sql_fn_param_ref(ParseState *pstate, ParamRef *pref)       |
| In prepared statement  | variable_paramref_hook(ParseState *pstate, ParamRef *pref) |
| In pl/sql function     | plpgsql_param_ref(ParseState *pstate, ParamRef *pref)      |

### how to handle input parameters.

First, parser will use an actual parameter to build a Const expression, and set the expression type based on its token type. Later analyzer will transform Const expression to Const node and use this list of nodes to build ParamListInfo so all the extern parameters can be passed to next stage.

```c
typedef struct ParamExternData
{
	Datum		value;			/* parameter value */
	bool		isnull;			/* is it NULL? */
	uint16		pflags;			/* flag bits, see above */
	Oid			ptype;			/* parameter's datatype, or 0 */
} ParamExternData;

typedef struct ParamListInfoData
{
	ParamFetchHook paramFetch;	/* parameter fetch hook */
	void	   *paramFetchArg;
	ParamCompileHook paramCompile;	/* parameter compile hook */
	void	   *paramCompileArg;
	ParserSetupHook parserSetup;	/* parser setup hook */
	void	   *parserSetupArg;
	char	   *paramValuesStr; /* params as a single string for errors */
	int			numParams;		/* nominal/maximum # of Params represented */

	/*
	 * params[] may be of length zero if paramFetch is supplied; otherwise it
	 * must be of length numParams.
	 */
	ParamExternData params[FLEXIBLE_ARRAY_MEMBER];
} ParamListInfoData;
```

Actually there are several functions to handle input parameters in different situations just like how analyzer handle Param node in above. 

- For a prepared statement, function EvaluateParams() will do this.  When user use EXECUTE command to execute a prepared statement, function EvaluateParams() will build a Const node list and transform input values to target type which stored in the cached info of prepared statements. Then use that param list  to build a ParamListInfo object. 
- For a sql function. In analyzer function tranformFuncCall() will pares the arguments in FuncCall node to a list of expression node, after the funcexpr node is transformed assign these expressoin nodes as its args. Before the function executed store these arguments in FunctionCallInfo so it can be fetched during execution. 

### How to assign input value to Param node

**In planner**

For a query with bind variables, planner may be the first place to fetch actual input value for Param node in query tree. For prepared statements, bind variables are also related to the choice of plan. When a prepared statement first time get executed, planner will build a plan for it and put the in cache. If this prepared statement do not have bind variables, planner will build a generic plan otherwise it will build a custom plan. Planner will try to optimize the plan by preprocessing those expression nodes in target list , WHERE clause, HAVING clause or a few others thing. During this process planner will try to simplify or evaluate  constant value. So a ‘Param’ node in planner may be optimized as a ‘Const’ node in function eval_const_expressions_mutator(). If it cannot be optimized, the node is retained in plan tree and it will be processed in executor. 

Like a following example, I created a very simple prepared statement to add two integers add set debug_print_plan to 'ON'. We can see the result, before executing the plan, the result has been calculated in the planner.

```
postgres=# prepare getsum(int,int) as select $1 + $2;
PREPARE
postgres=# set debug_print_plan to on;
SET
postgres=# execute getsum(2,1);
2021-04-06 11:20:42.545 CST [43871] LOG:  plan:
2021-04-06 11:20:42.545 CST [43871] DETAIL:     {PLANNEDSTMT 
	   :commandType 1 
	   :queryId 2821393787857181880 
	   :hasReturning false 
	   :hasModifyingCTE false 
	   :canSetTag true 
	   :transientPlan false 
	   :dependsOnRole false 
	   :parallelModeNeeded false 
	   :jitFlags 0 
	   :planTree 
	      {RESULT 
	      :startup_cost 0.00 
	      :total_cost 0.01 
	      :plan_rows 1 
	      :plan_width 4 
	      :parallel_aware false 
	      :parallel_safe false 
	      :plan_node_id 0 
	      :targetlist (
	         {TARGETENTRY 
	         :expr 
	            {CONST
	            :consttype 23 
	            :consttypmod -1 
	            :constcollid 0 
	            :constlen 4 
	            :constbyval true 
	            :constisnull false 
	            :location -1 
	            :constvalue 4 [ 3 0 0 0 0 0 0 0 ] <----------the result is 3
	            }
	         :resno 1 
	         :resname ?column? 
	         :ressortgroupref 0 
	         :resorigtbl 0 
	         :resorigcol 0 
	         :resjunk false
	         }
	      )
	      :qual <> 
	      :lefttree <> 
	      :righttree <> 
	      :initPlan <> 
	      :extParam (b)
	      :allParam (b)
	      :resconstantqual <>
	      }
	   :rtable (
	      {RTE 
	      :alias <> 
	      :eref 
	         {ALIAS 
	         :aliasname *RESULT* 
	         :colnames <>
	         }
	      :rtekind 8 
	      :lateral false 
	      :inh false 
	      :inFromCl false 
	      :requiredPerms 0 
	      :checkAsUser 0 
	      :selectedCols (b)
	      :insertedCols (b)
	      :updatedCols (b)
	      :extraUpdatedCols (b)
	      :securityQuals <>
	      }
	   )
	   :resultRelations <> 
	   :appendRelations <> 
	   :subplans <> 
	   :rewindPlanIDs (b)
	   :rowMarks <> 
	   :relationOids <> 
	   :partitionOids <> 
	   :invalItems <> 
	   :paramExecTypes <> 
	   :utilityStmt <> 
	   :stmt_location 0 
	   :stmt_len 41
	   }
	
2021-04-06 11:20:42.545 CST [43871] STATEMENT:  execute getsum(2,1);
 ?column? 
----------
        3
(1 row)
```

For prepared statements, bind variables are also related to the choice of plan. When a prepared statement first time get executed, planner will build a plan for it and put the in cache. If this prepared statement do not have bind variables, planner will build a generic plan otherwise it will build a custom plan.

**In executor**
At the initiation of the execution, the executor will generate execution steps according to the plan tree. If a Param node did' not get optimized in planner it will be passed to function ExecInitExprRec(). When the execution initialization function encounters a Param node with type PARAM_EXETERN, it will add an execution step of type EEOP_PARAM_EXTERN to the sequence of execution steps and set that step with parameter id and parameter type. Beside that all the extern parameters in ParamListInfo will be copied to query descriptor then it will be passed to so it can be fetched during later execution. 

Finally when execution start to run and executor start to evaluate expressions, it will call ExecInterpExpr() to do that. When this function reach a step with opcode EEOP_PARAM_EXTERN it will call ExecEvalParamExtern() function to fetch the corresponding id parameter which stored in execution context. There are some thing I did not mentioned yet, the paramFetch and ParamCompile functions in ParamListInfo, if they were set they will be called in executor. As far as I know, they works in plsql function. 

### Conclusion

Finally, we can see the code to handle parameter in Postgres is easily expandable. This blog just bring a rough introduction of it. Most of the examples in this blog come from prepared statements or sql functions, and there are some other scenarios where bind variables are used that have not been mentioned, and there are still many related details that have not been mentioned. You can learn more in the code.
