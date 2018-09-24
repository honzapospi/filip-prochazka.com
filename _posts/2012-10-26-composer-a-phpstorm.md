---
layout: blogpost
title: "Composer a PhpStorm"
permalink: blog/composer-a-phpstorm
date: 2012-10-26 11:30
tag: ["PHP", "PhpStorm", "IDE", "Composer"]
---

[Composer](http://getcomposer.org/) je **skvělý** nástroj na správu závislostí pro PHP. A [PhpStorm](http://www.jetbrains.com/phpstorm/) je docela kvalitní (ale hlavně rychlé) IDE. Když se sejdou dva takhle užitečné nástroje, někoho by napadlo, že by mohly spolupracovat.

O nativní podporu Composeru v PhpStormu [se již snažíme](http://youtrack.jetbrains.com/issue/WI-13046) a s trochou optimismu by to příští Vánoce mohlo být hotové ;) Ale někdo to prostě nevydrží a [podporu si přidá sám](https://twitter.com/vvondra/status/261553362405826561). Za nápad moc děkuji [Vojtěchovi](https://twitter.com/vvondra!)

Přes [External Tools](http://www.jetbrains.com/phpstorm/webhelp/external-tools.html) jde velice snadno vytvořit klikátka na externí nástroje.

![phpstorm-tools-composer](/content/phpstorm-tools-composer.png)

Které jdou spouštět z různých kontextových nabídek

![phpstorm-tools-composer1](/content/phpstorm-tools-composer1.png)

A výsledek operace zobrazí tak jako v konzoli

![phpstorm-tools-composer-run](/content/phpstorm-tools-composer-run.png)

Kde stažení: "phpstorm-tools.jar":/content/phpstorm-tools.jar (**File** > **Import Settings**)

Další zajímavý způsob integrace je použít "Command line tool support". [Více na PhpStorm blogu](http://blog.jetbrains.com/webide/2012/10/integrating-composer-command-line-tool-with-phpstorm/).

Jaké nástroje máte v PhpStormu (nebo v jiném IDE) podobně integrované vy?