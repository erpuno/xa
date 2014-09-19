
Distributed Transaction Coordinator
===================================

Sample

```erlang
-module(dtc).
-compile(export_all).

start() -> spawn(fun() -> coordinator() end).

coordinator() -> 
    receive
        {Result,State} -> io:format("Result: ~p with ~p",[Result,State]);
        ReqList when is_list(ReqList) -> spawn(fun() -> transaction(ReqList,self()) end);
        Req -> io:format("Unknown: ~p",[Req]) end, coordinator().

transaction(ReqList,Reply) -> map_reduce(ReqList,Reply,prepare).

map_reduce(ReqList,Reply,Command) ->
    Pid = spawn(fun() -> completion([],length(ReqList),Reply,Command) end),
    perform(ReqList,Pid,Command).
 
completion(State,Num,Reply,Command) ->
    receive
        {Node,Req,SqlType,Result,Command} -> 
            case {length(State) =:= Num,Command} of
                 {false,_}       -> completion([{Node,Req,SqlType,Result,Command}|State],Num,Reply,Command);
                 {true,commit}   -> Reply ! {ok,State};
                 {true,rollback} -> Reply ! {failed,State};
                 {true,prepare}  -> case lists:all(fun({_,_,_,{Answer,Id},_}) -> Answer =:= yes end,State) of
                                         true  -> map_reduce(State,self(),commit);
                                         false -> map_reduce(State,self(),rollback) end end end.

perform(State,Reply,Command) ->
  [ spawn(cohort(Node,SqlType,Req,Reply,Command)) || {Node,Req,SqlType,Result,_} <- State ].

cohort(Node,Req,SqlType,Reply,Command) ->
    {Answer,Parameter} = SqlType:Command(Node,Req),
    Reply ! {Node,Req,SqlType,{Answer,Parameter},Command}.
```

