# Munin auf einem Uberspace betreiben

In dieser Anleitung gehe ich nur Munin-Server ein und nicht auf dem Node.

## Installation

Zuerst wird der Quelltext auf dem Uberspace herunterladen und anschließend entpackt:

    mkdir ~/src
    cd ~/src
    wget https://github.com/munin-monitoring/munin/archive/stable-2.0.zip
    unzip stable-2.0
    cd munin-stable-2.0/

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

Damit Munin die Dateien direkt an den richtigen Ort installiert, werden noch zwei Symlinks erstellt:

    mkdir -p ~/opt/munin/www
    mkdir ~/html/munin/
    ln -s ~/html/munin/ ~/opt/munin/www/docs
    ln -s ~/cgi-bin/ ~/opt/munin/www/cgi

Nun kann Munin kompiliert und installiert werden:

    make
    make install

Damit Perl nun die von uns installierten Munin-Dateien und nicht die globalen von den Ubernauten benutzt
(diese sind inkompatibel mit unseren), müssen wir die Umgebungsvariable `PERL5LIB` um einen Pfad erweitern.
Das ganze legen wir direkt in der Datei `~/.bashrc` ab:

    echo "export PERL5LIB=/home/$USER/usr/local/share/perl5:\$PERL5LIB" >> ~/.bashrc

Damit die Änderung auch in der unseren aktuellen SSH-Session übernommen wird, muss die Datei `~/.bashrc` neu
gelesen werden:

    . ~/.bashrc

Munin benötigt außerdem noch eine reihe von  CPAN-Modulen, welche wir lokal installieren können. Dafür
greifen wir `local::lib` zurück. Wie du `local::lib` in deinem Uberspace installiert, wird im Uberspace-Wiki
erläutert: https://uberspace.de/dokuwiki/development:perl#lokale_cpan-module

Danach können die benötigen CPAN-Module installiert werden:

    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Time::HiRes)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Storable)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Digest::MD5)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(HTML::Template)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Text::Balanced)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Params::Validate)'
    # FIXME: nicht gefunden. Evtl. falscher Name?
    # perl -MCPAN -Mlocal::lib -e 'CPAN::install(TimeDate)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Net::SSLeay)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Getopt::Long)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(File::Copy::Recursive)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(CGI::Fast)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(IO::Socket::INET6)'
    perl -MCPAN -Mlocal::lib -e 'CPAN::install(Log::Log4perl)'

In die beiden Dateien `~/cgi-bin/munin-cgi-graph` und `~/cgi-bin/munin-cgi-html` muss direkt nach der
Shebang-Zeile folgendes hinzugefügt werden:

    use lib '/home/DEIN_USERNAME/usr/local/share/perl5';
    use local::lib;

In der Datei `~/etc/opt/munin/munin.conf` müssen die folgende Werte getroffen werden:

    graph_strategy cgi
    cgiurl_graph /cgi-bin/munin-cgi-graph
    html_strategy cgi

Jetzt muss die Datei `~/html/munin/.htaccess` angepasst werden.

    AuthUserFile /var/www/virtual/DEIN_USERNAME/html/munin/.htpasswd

Und folgendes am Ende hinzufügen:

    # Rewrites
    RewriteEngine On
    RewriteBase /munin
    
    # HTML
    RewriteRule ^$ index.html [L]
    RewriteCond %{REQUEST_URI} !^/munin/static
    RewriteCond %{REQUEST_URI} .html$
    RewriteRule ^(.*)          /cgi-bin/munin-cgi-html/$1 [L]

    # Images
    RewriteCond %{REQUEST_URI} !^static
    RewriteCond %{REQUEST_URI} .png$
    RewriteRule ^/(.*)         /cgi-bin/munin-cgi-graph/$1 [L]

Jetzt noch ein Benutzername/Kennwort für die HTTP-Authentifizierung von Munin bestimmt werden:

    htpasswd -m -c ~/html/munin/.htpasswd WUNSCHNAME
