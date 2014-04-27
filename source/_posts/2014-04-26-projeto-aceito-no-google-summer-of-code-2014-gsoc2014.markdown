---
layout: post
title: "Projeto aceito no Google Summer of Code 2014 (GSoC2014)"
date: 2014-04-26 12:24:51 -0300
comments: true
categories: [postgresql, unlogged, table, gsoc, google] 
---

Amigos, gostaria de compartilhar um pouco de minha emoção e alegria, pois [minha proposta de projeto](http://www.google-melange.com/gsoc/project/details/google/gsoc2014/fabriziomello/5738600293466112) foi aceita para o ["Google Summer of Code 2014"](http://www.google-melange.com/gsoc/homepage/google/gsoc2014).

Nos próximos meses estarei desenvolvendo uma nova funcionalidade para o [PostgreSQL](http://www.postgresql.org) financiado pelo [Google](http://www.google-melange.com/gsoc/homepage/google/gsoc2014), como projeto de pesquisa do curso de Pós-Graduação em [Tecnologias Aplicadas a Sistemas de Informação com Métodos Ágeis](http://www.uniritter.edu.br/pos/tecnologia/metodos_ageis) pela [Uniritter Porto Alegre/RS](http://www.uniritter.edu.br).

O principal objetivo deste projeto é permitir que ["unlogged tables"](http://www.postgresql.org/docs/current/static/sql-createtable.html) (tabelas que não geram registros no [WAL](http://www.postgresql.org/docs/current/static/wal-intro.html)) sejam transformadas em tabelas regulares (que geram registros no WAL) e vice-versa. Para que isso aconteça será adicionado mais duas cláusulas ao comando sql "ALTER TABLE":

	ALTER TABLE table_name SET LOGGED;
	ALTER TABLE table_name SET UNLOGGED;

Quem tiver interesse em acompanhar a realização do projeto pode visitar a [página do projeto no Wiki da comunidade PostgreSQL](http://wiki.postgresql.org/wiki/Allow_an_unlogged_table_to_be_changed_to_logged_GSoC_2014) e também o [código fonte no github](https://github.com/fabriziomello/postgres/tree/gsoc2014_alter_table_set_logged).
