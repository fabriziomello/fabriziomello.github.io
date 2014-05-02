---
layout: post
title: "PostgreSQL (aguardando pela 9.4) - Introduzindo JSONB, um formato estruturado para armazenar objetos JSON"
date: 2014-05-01 23:59:07 -0300
comments: true
categories: [postgresql, jsonb, depesz] 
---

**ATENÇÃO!** Este post é uma tradução para Português Brasil do [blog do Sr. Hubert Depesz Lubaczewski](http://www.depesz.com/2014/03/25/waiting-for-9-4-introduce-jsonb-a-structured-format-for-storing-json/). Fique a vontade para comentar quaisquer problemas na tradução.

No dia 23 de Março de 2014, Andrew Dunstan [commitou](http://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=d9134d0a355cfa447adc80db4505d5931084278a) o seguinte patch:

> **Introduzindo jsonb, um formato estruturado para armazenar objetos json.**
>
> O novo formato aceita exatamente os mesmos dados como o tipo json já existente.
> Entretanto este é armazenado em um formato que não requer a realização de um parse
> do text original para processá-lo, tornando-o muito mais adequado para indexação e
> outras operações. Espaços em brancos desnecessários são descartados, e a ordem das
> chaves dos objetos não é preservada. Nem são mantidas chaves de objetos duplicadas -
> o valor para uma determinada chave é armazenada uma única vez.
>
> O novo tipo possui todas as funções e operadores que o tipo json tem, com excessão
> das funções para geração de objetos json (to_json, json_agg, etc.), e também possui
> a mesma semântica. Adicionalmente existem classes de operadores para índices to tipo
> hash e btree, e duas classes para índices GIN o qual não existe para o tipo json.
>
> Essa funcionalidade evoluiu de um trabalho anterior do Oleg Bartunov e Teodor Sigaev,
> que se destinava a fornecer facilidades semelhantes ao hstore aninhado, mas que no
> final provou ter alguns problemas de compatibilidade significativos.
>
> Autores: Oleg Bartunov, Teodor Sigaev, Peter Geoghegan e Andrew Dunstan.
> Revisão: Andres Freund

Depois que ele foi commitado, ele foi explicado muito bem, mas eu decidi escrever sobre isso também, com alguns exemplos. Primeiro, vamos ver como ele funciona.

Vou começar com alguns valores de teste:

	{"a":"abc","d":"def","z":[1,2,3]}

	{"a":"abc","d";"def","z":[1x2,3]}

	{
		"a": "abc",
		"d": "def",
		"z": [1, 2, 3]
	}

	{"a":"abc","d":"def","z":[1,2,3],"d":"overwritten"}

Primeiro, vamos ver o que acontece após transformar esses valores em json e jsonb:

	select '{"a":"abc","d":"def","z":[1,2,3]}'::json;
	               json                
	-----------------------------------
	 {"a":"abc","d":"def","z":[1,2,3]}
	(1 row)

	select '{"a":"abc","d":"def","z":[1,2,3]}'::jsonb;
	                  jsonb                   
	------------------------------------------
	 {"a": "abc", "d": "def", "z": [1, 2, 3]}
	(1 row)

Tudo parece correto aqui, porém a saída do jsonb foi reformatada. Aqui ele não fez muita coisa, mas adicionou alguns espaços em branco. Isso é bom.

	select '{"a":"abc","d";"def","z":[1x2,3]}'::json;
	ERROR:  invalid input syntax for type json
	LINE 1: select '{"a":"abc","d";"def","z":[1x2,3]}'::json;
	               ^
	DETAIL:  Token ";" is invalid.
	CONTEXT:  JSON data, line 1: {"a":"abc","d";...

	select '{"a":"abc","d";"def","z":[1x2,3]}'::jsonb;
	ERROR:  invalid input syntax for type json
	LINE 1: select '{"a":"abc","d";"def","z":[1x2,3]}'::jsonb;
	               ^
	DETAIL:  Token ";" is invalid.
	CONTEXT:  JSON data, line 1: {"a":"abc","d";...

Em ambos os casos o erro foi reportado corretamente, mas no caso do jsonb ele disse: "invalid input syntax for type json". É provavelmente devido a ordem do cast, e normalmente isso deve estar correto. De qualquer maneira JSON e JSONB são similares o suficiente para não causar problemas.

	select '{
	    "a": "abc",
	    "d": "def",
	    "z": [1, 2, 3]
	}'::json;
	        json        
	--------------------
	 {                 +
	     "a": "abc",   +
	     "d": "def",   +
	     "z": [1, 2, 3]+
	 }
	(1 row)

	select '{
	    "a": "abc",
	    "d": "def",
	    "z": [1, 2, 3]
	}'::jsonb;
	                  jsonb                   
	------------------------------------------
	 {"a": "abc", "d": "def", "z": [1, 2, 3]}
	(1 row)

Aqui nós vemos a remoção de espaços em branco. Eu diria que é muito legal. É claro, a não ser que (por qualquer motivo) você queira manter os espaços, mas isso não deveria ser significativo, então dependendo de eles estarem lá não parece sensato.

	select '{"a":"abc","d":"def","z":[1,2,3],"d":"overwritten"}'::json;
	                        json                         
	-----------------------------------------------------
	 {"a":"abc","d":"def","z":[1,2,3],"d":"overwritten"}
	(1 row)

	select '{"a":"abc","d":"def","z":[1,2,3],"d":"overwritten"}'::jsonb;
	                      jsonb                       
	--------------------------------------------------
	 {"a": "abc", "d": "overwritten", "z": [1, 2, 3]}
	(1 row)

E o valor está sendo sobrescrito. Eu digo que isso é ótimo.

Quanto a utilização do espaço em disco, isso irá definitivamente depender do caso, mas um rápido teste mostra que jsonb pode utilizar bem mais espaço em disco:

	select pg_column_size('{"a":"abc","d":"def","z":[1,2,3]}'::jsonb);
	 pg_column_size 
	----------------
	             84
	(1 row)

	select pg_column_size('{"a":"abc","d":"def","z":[1,2,3]}'::json);
	 pg_column_size 
	----------------
	             37
	(1 row)

Por outro lado, para este JSON:

	{"widget": {
	    "debug": "on",
	    "window": {
	        "title": "Sample Konfabulator Widget",
	        "name": "main_window",
	        "width": 500,
	        "height": 500
	    },
	    "image": { 
	        "src": "Images/Sun.png",
	        "name": "sun1",
	        "hOffset": 250,
	        "vOffset": 250,
	        "alignment": "center"
	    },
	    "text": {
	        "data": "Click Here",
	        "size": 36,
	        "style": "bold",
	        "name": "text1",
	        "hOffset": 250,
	        "vOffset": 100,
	        "alignment": "center",
	        "onMouseUp": "sun1.opacity = (sun1.opacity / 100) * 90;"
	    }
	}}

O valor do JSON tem 605 bytes e o JSONb 524 bytes.

O que há de mais ...

Para JSONb temos mais operadores. Por exemplo - igualdade:

	select '{"a":1, "b":2}'::jsonb = '{"b":2, "a":1}'::jsonb;
	 ?column? 
	----------
	 t
	(1 row)

Mais operadores estão descritos na [documentação](http://www.postgresql.org/docs/devel/static/functions-json.html).

O que mais - o novo tipo de dados pode utilizar índices para pesquisar por elementos.

Eu criei uma tabela para testes:

	create table test (v jsonb);

e inseri nela 100 mil linhas, parecidas com:

	{"i": 42, "s": "ryzdaoop"}

Algumas das linhas (~ 1%) tem um valor adicional no json - chave "r" com valor 1.

Agora eu posso criar um índice nesta tabela:

	create index whatever on test using gin (v);
	CREATE INDEX

e agora:

	explain analyze select * from test where v ? 'r';
	                                                      QUERY PLAN                                                       
	-----------------------------------------------------------------------------------------------------------------------
	 Bitmap Heap Scan on test  (cost=16.77..307.23 rows=100 width=42) (actual time=0.554..2.670 rows=1024 loops=1)
	   Recheck Cond: (v ? 'r'::text)
	   Heap Blocks: exact=644
	   ->  Bitmap Index Scan on whatever  (cost=0.00..16.75 rows=100 width=0) (actual time=0.416..0.416 rows=1024 loops=1)
	         Index Cond: (v ? 'r'::text)
	 Planning time: 0.475 ms
	 Total runtime: 2.788 ms
	(7 rows)

Eu também poderia fazer:

	explain analyze select * from test where v @> '{"i":42}';
	                                                      QUERY PLAN                                                      
	----------------------------------------------------------------------------------------------------------------------
	 Bitmap Heap Scan on test  (cost=28.77..319.23 rows=100 width=42) (actual time=1.132..1.707 rows=103 loops=1)
	   Recheck Cond: (v @> '{"i": 42}'::jsonb)
	   Heap Blocks: exact=99
	   ->  Bitmap Index Scan on whatever  (cost=0.00..28.75 rows=100 width=0) (actual time=1.089..1.089 rows=103 loops=1)
	         Index Cond: (v @> '{"i": 42}'::jsonb)
	 Planning time: 0.482 ms
	 Total runtime: 1.783 ms
	(7 rows)

Ou mesmo:

	explain analyze select * from test where v @> '{"i":42, "r":1}';
	                                                     QUERY PLAN                                                     
	--------------------------------------------------------------------------------------------------------------------
	 Bitmap Heap Scan on test  (cost=52.77..343.23 rows=100 width=42) (actual time=1.171..1.191 rows=3 loops=1)
	   Recheck Cond: (v @> '{"i": 42, "r": 1}'::jsonb)
	   Heap Blocks: exact=3
	   ->  Bitmap Index Scan on whatever  (cost=0.00..52.75 rows=100 width=0) (actual time=1.143..1.143 rows=3 loops=1)
	         Index Cond: (v @> '{"i": 42, "r": 1}'::jsonb)
	 Planning time: 0.530 ms
	 Total runtime: 1.256 ms
	(7 rows)

Isso é muito legal.

Infelizmente você não pode utilizar índices para realizar uma pesquisa mais "profunda". O que significa isso?

Vamos supor que você tem um valor JSON como este:

	{"a": [1,2,3,4]}

Você pode indexá-lo, e pesquisar por "value ? 'a'", ou "value @> '{"a":[1,2,3,4]}'", mas você não pode pesquisar (utilizando índice) por "linhas que tenha 3 na matriz que está sob a chave 'a' no json".

É claro que você pode trabalhar em torno dele criando índice em ((value -> 'a')), se for isso que você realmente precisa.

De qualquer maneira eu realmente gostei disso. Parece que funciona muito bem, e eu espero que tenhamos mais recursos no futuro.

Valeu pessoal.
