---
layout: blogpost
title: "NewRelic: monitoring aplikace na Nette Frameworku"
permalink: blog/newrelic-monitoring-aplikace-na-nette-frameworku
date: 2013-03-22 20:30
tag: ["Nette Framework", "NewRelic", "Monitoring", "PHP"]
---

Má Vaše aplikace víc než pět návštěv denně? Pak není od věci nějakým způsobem monitorovat, co se děje. Za tímhle účelem vznikají nejrůznější placené i opensource řešení. Některé lepší, některé horší. Několik měsíců zpátky jsem řešil, jaký monitoring nasadím na svoje sexy [VPS od Wedosu](https://hosting.wedos.com/cs/virtualni-servery.html) (na kterém běží i tento blog).

<blockquote class="twitter-tweet" lang="cs"><p><a href="https://twitter.com/wedoscom">@wedoscom</a> Monitoring! To je jediná věc, která mi chybí! Nepotřebuju návštěvnost, ani analytiku - stačí vytíženost zdrojů na VPS.</p>&mdash; Filip Procházka (@ProchazkaFilip) <a href="https://twitter.com/ProchazkaFilip/statuses/214754921571041280">June 18, 2012</a></blockquote>

Nebudu to protahovat, zvolil jsem nakonec [NewRelic](https://newrelic.com/), který mi poradil [Honza Doleček](https://twitter.com/juznacz). Jeho jediné mínus je, že je docela drahý. Ale po měsíci trial verze a tričku zdarma už se mi nechtělo nikam migrovat.

Pak jsem dlouho monitoring neřešil a teď máme NewRelic i v [Damejidlo.cz](https://www.damejidlo.cz/). Na své VPS mám pár malých webíků, ale tady už začíná být kritické, mít vše pod dohledem.

Před pár dny jsem objevil [killer feature NewRelicu](https://newrelic.com/docs/php/the-php-api) a o tu bych se s Vámi chtěl zde podělit. Je to jeho PHP API. Abych to trošku rozvedl, NewRelic se instaluje tak, že [do phpčka zavedete modul a nastavíte IDčko aplikace](https://newrelic.com/docs/php/quick-installation-instructions-advanced-users). V ten moment začne rozsíření odesílat data na jejich servery a já se můžu kochat krásnými grafy :)

> ![newrelic-overview](/content/newrelic-overview.png)
>
> <small>A to prý je Doctrine2 pomalá ;)</small>

[Stejným způsobem, prostou instalací balíku](https://newrelic.com/docs/server/server-monitor-installation-ubuntu-and-debian#apt), jde rozchodit i monitoring systému.

> [![newrelic-pizza-overview](/content/newrelic-pizza-overview.png)](/content/newrelic-pizza-overview.png)
>
> <small>Náš dev server se moc nenadře :)</small>


## S chytrými frameworky je ale problém..

a určitě Vám hned dojde jaký, při pohledu na tento screenshot.

> ![newrelic-index](/content/newrelic-index.png)
>
> <small>NewRelic nerozlišuje adresy, protože všechno jde na index.php</small>

Další "problém" je, že Nette Framework sám řeši všechny chyby a výjimky a NewRelic se tak vůbec v tomto nedostane ke slovu. Když tedy server začne spamovat logy laděnkama, dozvím se to až když mi přijde email, který ani navíc přijít nemusí.

A tady přichází k řeči PHP API, na které mě též upozornil [Honza](https://twitter.com/juznacz). Chtěl jsem to mít cool, tak jsem řešení založil "na svém Event systému":/blog/eventy-a-nette-framework.

Tohle API pokrývá rozlišení jednotlivých requestů, logování errorů a výjimek a umí taky označit požadavek jako "proces na pozadí"((background job)). Což není špatné rozlišit, protože vydatně používáme [CLI scripty přes Symfony Consoli](https://github.com/kdyby/console), pouštené cronem a nechceme, aby se nám pletly do frontend requestů. Byť snad všechny zatím roztřídil správně, jistota je jistota :)

Kompletní řešení tedy vypadá takto

~~~ php
namespace NewRelic;

use Kdyby;
use Nette;
use Nette\Application\Application;
use Nette\Application\Request;
use Nette\Diagnostics\Debugger;

class NewRelicProfilingListener extends Nette\Object implements Kdyby\Events\Subscriber
{
    public function getSubscribedEvents()
    {
        return array(
            'Nette\\Application\\Application::onStartup',
            'Nette\\Application\\Application::onRequest',
            'Nette\\Application\\Application::onError'
        );
    }

    public function onStartup(Application $app)
    {
        if (!extension_loaded('newrelic')) {
            return;
        }

        // registrace vlastního loggeru na errory
        Debugger::$logger = new Logger;
        Debugger::$logger->directory =& Debugger::$logDirectory;
        Debugger::$logger->email =& Debugger::$email;
    }

    public function onRequest(Application $app, Request $request)
    {
        if (!extension_loaded('newrelic')) {
            return;
        }

        if (PHP_SAPI === 'cli') {
            // uložit v čitelném formátu
            newrelic_name_transaction('$ ' . basename($_SERVER['argv'][0]) . ' ' . implode(' ', array_slice($_SERVER['argv'], 1)));

            // označit jako proces na pozadí
            newrelic_background_job(TRUE);

            return;
        }

        // pojmenování požadavku podle presenteru a akce
        $params = $request->getParameters();
        newrelic_name_transaction($request->getPresenterName() . (isset($params['action']) ? ':' . $params['action'] : ''));
    }

    public function onError(Application $app, \Exception $e)
    {
        if (!extension_loaded('newrelic')) {
            return;
        }

        if ($e instanceof Nette\Application\BadRequestException) {
            return; // skip
        }

        // logovat pouze výjimky, které se dostanou až k uživateli jako chyba 500
        newrelic_notice_error($e->getMessage(), $e);
    }
}
~~~

A ještě `Logger`

~~~ php
namespace NewRelic;

use Nette;

class Logger extends Nette\Diagnostics\Logger
{
    public function log($message, $priority = self::INFO)
    {
        $res = parent::log($message, $priority);

        // pouze zprávy, které jsou označené jako chyby
        if ($priority === self::ERROR || $priority === self::CRITICAL) {
            if (is_array($message)) {
                $message = implode(' ', $message);
            }
            newrelic_notice_error($message);
        }

        return $res;
    }
}
~~~

Pokud máte v aplikaci [Kdyby/Events](https://github.com/kdyby/events), tak je rozběhání listeneru otázkou tří řádků konfigurace

~~~ neon
services:
    newRelicListener:
        class: NewRelic\NewRelicProfilingListener
        tag: [kdyby.subscriber]
~~~

Co se týče logování chyb, má NewRelic [do laděnky](https://doc.nette.org/cs/debugging#toc-vizualizace-chyb-a-vyjimek) ještě světelné míle daleko. To co tam je teď, připomíná spíše brášku `log/error.log`. Pro laděnky si tedy stále musím dojít do logu na server. Je ale super vidět prolnutí chybovosti vzhledem k počtu požadavků. Určitě tedy stojí za to, posílat chyby do NewRelicu.

Na druhou stranu, chudý rozbor chyb je vynahrazený luxusním profilerem, který automaticky loguje requesty, které trvají déle než by měly a velice přesně, až možná doterně, upozorňuje na úzká hrdla aplikace.


## Výsledek

Nyní už uvidím jednotlivé requesty podle presenteru a akce

> ![newrelic-presenters](/content/newrelic-presenters.png)
>
> <small>Requesty seskupené podle presenterů</small>

stejně tak procesy, které probíhají na pozadí

> ![newrelic-background-jobs](/content/newrelic-background-jobs.png)
>
> <small>Procesy na pozadí</small>

a také všechny chyby, které se v aplikaci vyskytnou.

> ![newrelic-errors](/content/newrelic-errors.png)
>
> <small>Procento chyb vzhledem k requestům</small>

Naštěstí jich tam moc není :)

Za mě můžu NewRelic jedině doporučit. U větších aplikací je to must have. Jak monitorujete svoje aplikace vy?
