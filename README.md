# Munin auf einem Uberspace betreiben

In dieser Anleitung gehe ich nur Munin-Server ein und nicht auf dem Node. Diese Anleitung beschreibt also, wie man andere Server und nicht den Uberspace-Host mit Munin überwacht.

## Vorbereitung

Munin benötigt außerdem noch eine reihe von Perl-Modulen, welche wir lokal installieren können. Dafür
greifen wir auf `local::lib` zurück. Wie du `local::lib` in deinem Uberspace installiert, wird im Uberspace-Wiki
erläutert: https://uberspace.de/dokuwiki/development:perl#lokale_cpan-module
Das Perl-Modul `RRDs.pm` benötigt "etwas" mehr Arbeit: Es benötigt rrdtool, welches widerum pango benötigt:

    toast arm http://ftp.acc.umu.se/pub/GNOME/sources/pango/1.28/pango-1.28.0.tar.gz
    
Für die generierung der Grafiken per `cron`-Job brauchen wir noch folgenden Befehl:

    pango-querymodules > '/home/DEIN_USERNAME/.toast/armed/etc/pango/pango.modules'

Wie toast funktioniert kann hier nachgelesen werden: https://wiki.uberspace.de/system:toast

Und und so installert man dann rrdtool:

    cd ~/src/
    wget http://oss.oetiker.ch/rrdtool/pub/rrdtool.tar.gz
    tar xzf rrdtool.tar.gz
    cd rrdtool-*/
    export PKG_CONFIG_PATH=/home/$USER/.toast/armed/lib/pkgconfig/
    ./configure --prefix=/home/$USER/
    make
    make install 

Die anderen Perl-Module können bequemer installiert werden:

    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Time::HiRes)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Storable)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Digest::MD5)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(HTML::Template)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Text::Balanced)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Params::Validate)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Getopt::Long)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(File::Copy::Recursive)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(IO::Socket::INET6)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Log::Log4perl)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Net::SSLeay)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(CGI::Fast)'

Das Perl-Modul `Date::Manip` kann leider nicht auf diese Weise installiert werden. Die Installation wird Aufgrund von hohem Speicherverbrauch von uberspace aus, beendet.
Als Alternative Methode habe ich vom uberspace Support folgende Methode empfohlen bekommen. Einfach mittels Perlbrew eine lokale Instanz erzeugen:

    perlbrew lib create perl-5.14.2@local
    perlbrew switch perl-5.14.2@local

Und dann anschließend den normalen Befehl verwenden:

    cpan Date::Manip
    cpan Net::SNMP

Jetzt ist das Modul unter `/home/$USER/.perlbrew/libs/perl-5.14.2@local/lib/perl5` installiert.
Mehr Informationen zu perlbrew findest du unter: http://perlbrew.pl

Damit Munin kompliliert und installiert werden kann, muss perlbrew ausgeschaltet werden:
    
    perlbrew switch-off

## Installation

Jetzt kann es mit Munin losgehen. Zuerst wird der Quelltext auf dem Uberspace herunterladen und anschließend entpackt:

    cd ~/src/
    wget http://downloads.munin-monitoring.org/munin/stable/2.0.25/munin-2.0.25.tar.gz
    tar xzf munin-2.0.25.tar.gz
    cd munin-2.0.25/

Jetzt wird eine Variable definiert, welche beim Kompilieren von Munin benutzt wird und das Ziel für die Installation bestimmt.

    export DESTDIR=/home/$USER

In der Datei `Makefile.config` müssen wir bei Zeile 120 noch ein paar Benutzernamen durch unseren eigenen ersetzen:

    # User to run munin as
    USER       := DEIN_USERNAME
    GROUP      := DEIN_USERNAME

    # Default user to run the plugins as
    PLUGINUSER := DEIN_USERNAME

    # Default user to run the cgi as
    CGIUSER := DEIN_USERNAME

Jetzt muss noch die Datei `Makefile` angepasst werden. Die Zeile 147 kann auskommentiert werden:

    $(CHOWN) root:root $(PLUGSTATE)

Bitte prüfe kurz ob die Softlinks `~/html` und `~/cgi-bin` korrekt in deinem uberspace auf `/var/www/virtual/DEIN_USERNAME/html` bzw.  `/var/www/virtual/DEIN_USERNAME/cgi-bin` eingerichtet sind. 

Damit Munin die Dateien direkt an den richtigen Ort installiert, werden noch zwei Symlinks erstellt:

    mkdir -p ~/opt/munin/www
    mkdir -p ~/html/munin
    ln -s ~/html/munin ~/opt/munin/www/docs
    ln -s ~/cgi-bin ~/opt/munin/www/cgi

Nun kann Munin kompiliert und installiert werden:

    make
    make install

Damit Perl nun die von uns installierten Munin-Dateien und nicht die globalen von den Ubernauten benutzt
(diese sind inkompatibel mit unseren), müssen wir die Umgebungsvariable `PERL5LIB` um einen Pfad erweitern.
Das ganze legen wir direkt in der Datei `~/.bashrc` ab:

    echo "export PERL5LIB=/home/$USER/usr/local/share/perl5:/home/$USER/lib/perl:\$PERL5LIB" >> ~/.bashrc

Damit die Änderung auch in der unseren aktuellen SSH-Session übernommen wird, muss die Datei `~/.bashrc` neu
gelesen werden:

    . ~/.bashrc

## Anpassungen an der Installation

In die beiden Dateien `~/html/munin/cgi-bin/munin-cgi-graph` und `~/html/munin/cgi-bin/munin-cgi-html` muss direkt nach der
Shebang-Zeile folgendes hinzugefügt werden:

    use lib '/home/DEIN_USERNAME/usr/local/share/perl5';
    use lib '/home/DEIN_USERNAME/lib/perl';
    use local::lib;

In der Datei `~/etc/opt/munin/munin.conf` müssen die folgende Werte getroffen werden:

    graph_strategy cron
    cgiurl_graph /munin/cgi-bin/munin-cgi-graph
    html_strategy cgi

Außerdem muss in dieser Datei der oder die zu überwachenden Server eingetragen werden.

## Konfiguration für die Website

Jetzt muss die Datei `~/html/munin/.htaccess` erstellt werden:

    AuthUserFile /var/www/virtual/DEIN_USERNAME/.htuser
    
    # Rewrites
    RewriteEngine On
    RewriteBase /munin
    
    # HTML
    RewriteRule ^$ index.html [L]
    RewriteCond %{REQUEST_URI} !^/munin/static
    RewriteCond %{REQUEST_URI} !^/munin/cgi-bin
    RewriteCond %{REQUEST_URI} .html$
    RewriteRule ^(.*)          /cgi-bin/munin-cgi-html/$1 [L]
    
    # Images
    RewriteCond %{REQUEST_URI} !^static
    RewriteCond %{REQUEST_URI} .png$
    RewriteRule ^/(.*)         /cgi-bin/munin-cgi-graph/$1 [L]

Jetzt noch ein Benutzername/Kennwort für die HTTP-Authentifizierung von Munin bestimmt werden:

    htpasswd -m -c /var/www/virtual/DEIN_USERNAME/.htuser WUNSCHNAME

## Konfiguration des `cron` Jobs

Nun fehlt noch ein runwhen-Job, welcher alle 5 Minuten `~/opt/munin/bin/munin-cron` aufruft

    uberspace-setup-svscan
    runwhen-conf ~/etc/run-munin-cron  ~/opt/munin/bin/munin-cron

In der Datei `~/etc/run-munin-cron/run` der Variable `RUNWHEN` den Wert `,M/5` geben,

    RUNWHEN=",M/5"

sowie folgendes in Zeile 29 schreiben:

    eval $(perl -I$HOME/perl5/lib/perl5 -Mlocal::lib)
    export PERL5LIB=/home/$USER/.perlbrew/libs/perl-5.14.2@local/lib/perl5:/home/$USER/usr/local/share/perl5:/home/$USER/lib/perl:$PERL5LIB

Jetzt noch den Symlink erstellen, damit der runwhen-Job auch gestartet wird:

    ln -s /home/$USER/etc/run-munin-cron ~/service/munin-cron

Und so startest du den Job:
    
    svc -u ~/service/munin-cron
    
Mehr zum Thema `run-when` findest du hier bei uberspace https://wiki.uberspace.de/system:runwhen

## Analyse / Logs

Auf der Seite zu den `daemontools` (https://wiki.uberspace.de/system:daemontools) gibt es noch ein nützliches `readlog`Skript. Wenn du das zur `.bashrc`hinzufügt kannst du mit 

    readlog munin-cron
    
herausfinden ob alles sauber eingerichtet ist.

Wenn irgendwas nicht funktioniert, lohnt sich auch ein Blick in die munin-Logs im Verzeichnis `~/opt/munin/log/munin`. In der Datei `munin-update.log` stehen Fehlermeldung im Zusammenhang mit dem Abruf der Daten vom munin-Node.

## Fertig...

Nun kann man über `http://DEIN_USERNAME.HOST.uberspace.de/munin/` Munin aufrufen. Solange Munin noch keine Daten gesammelt hat, wird es zu einem Fehler kommen.
Ebenso bricht `munin-cron` ab, wenn es keine Daten sammeln konnte.

## Bonus: eigene Domain

Wenn du eine eigene Domain hast, kannst du diese mit diesem Befehl dem uberspace hinzufügen:

    uberspace-add-domain -d munin.example.com -w
    
(Nicht vergessen entsprechende Einträge mit der IP-Adresse im DNS Zone File zu machen.)
Als nächstes lege einfach den Ordner `munin.example.com`im Ordner `/var/www/virtual/DEIN_USERNAME/` an. Kopiere einfach die oben erstellte `.htaccess` in diesen Ordner. In der `.htaccess` müssen die Zeilen 9 und 10 so angepasst werden:

    RewriteCond %{REQUEST_URI} !^/static
    RewriteCond %{REQUEST_URI} !^/cgi-bin

(Der Ordner `static` wird automatisch angelegt.)

Zu guter letzt muss noch der Softlink für die Grafiken geändert werden:

    rm ~/opt/munin/www/docs
    ln -s /var/www/virtual/DEIN_USERNAME/munin.example.com ~/opt/munin/www/docs

Jetzt sollte die munin Installation unter `http://munin.example.com` erreichbar sein. (Es dauert höchsten 5 Minuten bis die Bilder angezeigt werden und eventuell zwei Aufrufe bis die Seite komplett aussieht.)

## Bonus II: Daten umziehen

Angenommen du willst deine munin Installation auf einen anderen uberspace-Account umziehen. Dazu empfehle ich diese Anleitung für den Ziel-uberspace-Account zuerst durchzuführen. Wenn die Daten im neuen Account korrekt ermittelt und angezeigt werden können die Daten folgendermaßen vom alten auf den neuen uberspace-Account kopiert werden.

Hinweis: Für lückenlose Daten muss das packen, kopieren und entpacken zwischen zwei `cron`-Jobs ausgeführt werden.

Hinweis 2: Die Beschreibung gilt nur für umzüge, **nicht** für das Zusammenführen von zwei Datenbeständen!

1. Backup auf dem **neuen** uberspace erstellen:

        [NEW_USERNAME@HOST2 ~]$ cd ~/var/opt/
        [NEW_USERNAME@HOST2 opt]$ tar -czf munin_backup.tar.gz munin

2. Datenpaket auf dem **alten** uberspace erstellen:

        [OLD_USERNAME@HOST1 ~]$ cd ~/var/opt/
        [OLD_USERNAME@HOST1 opt]$ tar -czf munin_now.tar.gz munin

3. Datenpaket auf dem **neuen** uberspace holen und entpacken:

        [NEW_USERNAME@HOST2 opt]$ scp OLD_USERNAME@HOST1.uberspace.de:~/var/opt/munin_now.tar.gz . 
        [NEW_USERNAME@HOST2 opt]$ tar xvfz munin_now.tar.gz 
    
4. Auf dem **neuen** uberspace die per `cron`-Job erstellten png-Dateien löschen (werden mit dem nächsten Job wieder neu erstellt.)

        [NEW_USERNAME@HOST2 opt]$ rm -rf ~/opt/munin/www/docs/DEINE_DOMAIN/*
    
Nach spätestens 5 Minuten sollten dann auf dem neuen uberspace die Daten vom alten uberspace angezeigt werden.
