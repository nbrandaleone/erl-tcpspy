# erl-tcpspy

# Another erlang proxy, from a gist
-module(generic_proxy).

-export([run/0]).

-define(PORT_FROM, 63790).
-define(PORT_TO, 6379).
-define(BACKLOG, 10000).

run() ->
  {ok, Socket} = gen_tcp:listen(0, [
    {backlog, ?BACKLOG}, {reuseaddr, true},
    {port, ?PORT_FROM}, binary, {packet, 0}, {active, true}
  ]),
  accept(Socket).

accept(Socket) ->
  {ok, ClientSocket} = gen_tcp:accept(Socket),
  Handler = spawn(fun() ->
    receive
      go -> go
    end,
    {ok, TargetSocket} = gen_tcp:connect("127.0.0.1", ?PORT_TO, [
      binary, {nodelay, true}, {packet, 0}, {active, true}
    ]),
    loop(TargetSocket, ClientSocket)
  end),
  gen_tcp:controlling_process(ClientSocket, Handler),
  Handler ! go,
  accept(Socket).

loop(TargetSocket, ClientSocket) ->
  Continue = receive
    {tcp, TargetSocket, Data} -> gen_tcp:send(ClientSocket, Data), true;
    {tcp, ClientSocket, Data} -> gen_tcp:send(TargetSocket, Data), true;
    {tcp_closed, _} ->
      gen_tcp:close(TargetSocket),
      gen_tcp:close(ClientSocket),
      false;
    X ->
      io:format("Unknown msg: ~p", [X]),
      gen_tcp:close(TargetSocket),
      gen_tcp:close(ClientSocket),
      false
  end,
  case Continue of
    true -> loop(TargetSocket, ClientSocket);
    false -> ok
  end.

