# Java for Sysadmins

You have been handed a WAR file and told to run it. Nobody told you what a WAR is, why the server that runs it has two home directories, or why the documentation reads like it was written for someone else. It was. This is the translation.

## Why this document exists

Java has its own vocabulary for things you already know, and the words point at the wrong Linux concepts. "Home" is not `$HOME`. "Container" is not Docker. "Module" is not a kernel module. "Context" is not a kubectl context. Once each word is translated once, the model underneath is ordinary: a program, a config directory, a plugin list and an application. The people who know this never had to explain it to an outsider, so nobody wrote it down.

## A short history, with Sun in mind

Sun Microsystems released Java in 1995 and bet the company on it. The name went on everything: phones, smart cards, browsers, servers. Three sizes of the same product emerged:

- **Java SE**, Standard Edition: the language, the virtual machine and the standard library. What `java` on your server is.
- **Java EE**, Enterprise Edition: a set of specifications layered on top of SE for the things every server application needs. HTTP handling, transactions, messaging, database mapping, remoting.
- **Java ME**, Micro Edition: phones and set-top boxes. Dead.

Java EE was specifications, not code. Sun wrote the contract, wrote a reference implementation to prove the contract could be met, wrote a compatibility test kit, and licensed the "J2EE compatible" brand to vendors: IBM WebSphere, BEA WebLogic, Oracle, later JBoss. Sun made money from the brand and from the Solaris and SPARC hardware underneath, not from the software. The buyers were banks and telcos, the same buyers COBOL had, through the same vendors, with the same promise: certify once, never rewrite. The platform therefore stopped changing, and Java in the enterprise became the thing it was supposed to replace.

In 2017 Oracle, which had bought Sun in 2010, handed Java EE to the Eclipse Foundation but kept the trademarks on "Java" and on the `javax` package prefix. Eclipse renamed the platform **Jakarta EE**. Jakarta EE 8 is Java EE 8 verbatim. Jakarta EE 9 renamed every package from `javax.*` to `jakarta.*` with no behavioural change. Jakarta EE 10 is where new features resumed. Every application and every server had to pick a side of that rename, and you will see the seam in the software you run.

One more collision: "Jakarta" was also the name of Apache's umbrella project for Java code from 1999 to 2011. Tomcat, Ant and Commons were born there. Eclipse reused the name in 2018 with Apache's permission. When you see "jakarta" today, it means the Eclipse one.

## The JVM, and what you are actually running

Java source compiles to **bytecode**, an instruction set for a machine that does not exist. The **JVM**, Java Virtual Machine, is the program that executes it: it loads `.class` files, verifies them, interprets the bytecode, then compiles the hot paths to native code at runtime (the JIT, just-in-time compiler) and manages memory with a garbage collector. `java` on your server is the JVM plus the standard library. When you `ps`, you see one `java` process per application server, and that process is the JVM; Tomcat, Jetty and the WAR are all just classes loaded inside it.

Two consequences matter operationally.

Memory is not what `free` tells you. The JVM reserves a **heap** for objects, sized by `-Xms` (initial) and `-Xmx` (maximum), and will happily use all of `-Xmx` before collecting, so a JVM sitting at its maximum heap is not leaking, it is behaving as configured. On top of the heap sit thread stacks, JIT code cache, metaspace for loaded classes and native buffers, so the memory the process actually occupies (the RES column in `top`) is the heap plus a few hundred megabytes. Since JDK 10 the default `-Xmx` is one quarter of the machine's memory. The JVM reads that figure from the cgroup it runs in, not from `/proc/meminfo`. On bare metal or a VM, that is the total RAM of the machine. Inside a Docker container started with `--memory=2g`, that is 2 GB, so the default heap is 512 MB. Before JDK 10 the JVM read `/proc/meminfo` instead. Java 8 behaved the same way until a late 2018 update. On a 64 GB host the JVM saw the full memory and sized its heap to 16 GB (the aforementioned quarter). It did not read the cgroup, so it had no idea a 2 GB limit existed and planned as if it had 16 GB to fill. The kernel enforced the limit regardless: the moment the heap grew past 2 GB the OOM killer shot the process. That is why every old Java-in-Docker guide tells you to set `-Xmx` by hand. On a 16 GB server dedicated to one application that leaves 12 GB unused, so always set `-Xmx` explicitly; a common starting point is half to three quarters of the machine, leaving room for the non-heap parts of the process and the OS page cache.

Startup is slow and then fast. The interpreter runs first, the JIT kicks in after thousands of invocations, so a JVM is at its slowest in its first minute and at its fastest after an hour. Restarting to deploy throws that away every time, which is one more reason the one-WAR-per-JVM practice hurts.

Names you will meet:

- **JDK**, Java Development Kit: the compiler plus the runtime. The package you install.
- **JRE**, Java Runtime Environment: the runtime alone. No longer shipped separately, so you install a JDK even to run things. The `-headless` package variants leave out the desktop libraries (AWT, Swing, fonts, printing), which are still part of the standard runtime because they were never removed, not because servers need them; the full package pulls in X11 and font dependencies for nothing. On a server, headless is the normal package.
- **OpenJDK**: the open-source codebase. Several organisations build it into binaries: Oracle, Red Hat (the distro packages on EL), Eclipse Temurin (the Eclipse Foundation's neutral build, formerly AdoptOpenJDK), Amazon Corretto (Amazon's build of the same OpenJDK source, made because Amazon runs enormous amounts of Java internally and wanted to control its own patches; it is what you get by default on AWS machine images), Azul Zulu (a company that sells Java support), Microsoft. All pass the same compatibility kit and differ in support terms and patch cadence, not behaviour. The names matter only when you download a tarball; the distro package is Red Hat's or Debian's build and you never see them.
- **Version numbers**, by epoch:
  - Sun, 1996 to 1998: 1.0, 1.1.
  - Sun, "Java 2", 1998 to 2006: 1.2, 1.3, 1.4, 5 (called 1.5 internally), 6. This is where the J2SE and J2EE names come from.
  - Sun then Oracle, 2006 to 2014: 7 (2011, first Oracle release), 8 (2014). Still numbered 1.7 and 1.8 in `java -version` output.
  - Oracle, six-month cadence, 2017 onward: 9, 10, 11, 12 and so on, one every March and September. Every fourth is long-term support: 11 (2018), 17 (2021), 21 (2023), 25 (2025). The rest are dead six months after release.
  - Java 8 is the exception: it came out in 2014, but it is still supported by Red Hat and Azul and still in production everywhere. It was the last release before the Java 9 compatibility break, so it is the last version an old application runs on without its dependencies being updated first, and enough customers pay for that to keep the security patches coming. The full story is under "Why old applications refuse to run on a new JDK" below.
- **Target version**: applications declare the minimum JDK they compile against. Spring Boot 2 era applications want 11 or 17, Jetty 12 wants 17.

### Which JDK to install

The distribution package, `java-17-openjdk-headless` on EL, `openjdk-17-jre-headless` on Debian. It is the same OpenJDK, patched by the distro, updated by `dnf` or `apt` with everything else, and inside the distro's security process. A vendor tarball is for two cases only: a version the distribution does not ship, or a container image with a specific JDK baked in. Pick an LTS version and never a six-month release or an early-access build: vendors support only LTS, ship security updates quarterly for those, and reserve the right to ship nothing for anything else. Do not start anything new on 8; it is 2014 code kept alive on paid support.

### Why old applications refuse to run on a new JDK

Java 9 introduced the module system and Java 11 removed the Java EE pieces that had been bundled with the runtime since the 2000s (XML binding, SOAP, activation, CORBA). An application compiled against 8 that used them starts on 11 and dies with `ClassNotFoundException: javax.xml.bind.JAXBContext` or similar. That is the whole of the 8 to 11 pain: not the language, the missing jars. Applications fixed it by adding those libraries as ordinary dependencies, and anything maintained since 2019 has done so; anything that has not is telling you something about its maintenance plan.

### Why Oracle did that

Sun's rule had been never to remove anything, so the runtime grew for twenty years into one monolithic jar, with libraries reaching freely into its internals, and nothing inside it could ever change. Java 9 put a fence around the internals so the JDK could evolve. Java 11 removed the Java EE libraries from the runtime. Those libraries had originally been developed as part of Java EE, and in the 2000s Sun had copied them into the standard runtime for convenience; the copies then fell behind and stopped being maintained while still sitting in the main Java tree. Once Java EE moved to Eclipse and those libraries were being developed again, the JDK dropped its stale copies and applications were told to depend on the current versions like any other library. Java 8 stayed in production for ten years because:

1. Java 9 broke compatibility, so moving off 8 meant updating every framework, library and agent an application used, not just the JDK.
2. Oracle ended free updates for 8 in 2019 and at the same time moved to a release every six months. The cadence change was a reaction to Java 9 itself: it had taken three and a half years and slipped twice because every feature waited for the module system, so Oracle decided releases would ship on a fixed date with whatever was ready, and mark every fourth one as long-term support. Sensible for the language, but organisations read the two changes together as "Java is now a subscription with a moving target" and froze on 8 rather than chase that target.
3. Java 8 was good enough. Lambdas and streams were the last language features most application code needed, and it had four stable years before the break.

Applications that ran unchanged from Java 5 to Java 8 could not move without their whole dependency tree moving first, and nobody forced the issue.

Flags go on the `java` command line. `-X` flags are standard across JVMs (`-Xmx`), `-XX:` flags are HotSpot-specific and change between versions (`-XX:+UseG1GC`), `-D` sets system properties the application reads (`-Duser.timezone=UTC`). Where those flags live is a per-server question, and one of the things that separates Tomcat from Jetty below.

## Servlets, JSP and the WAR

A **servlet** is a Java class that the server instantiates once and calls for every HTTP request. The server owns the sockets, the threads and the lifecycle; the class gets a request object and a response object. In 1996 the competing standard was CGI. CGI forked a process per request, and on the CPUs of the time forking a JVM per request took seconds, far from ideal. A servlet is loaded once and stays resident, so the JVM startup cost is paid once. One instance serves all requests concurrently. In software engineering terms, a servlet is effectively a singleton and its instance fields are shared state.

The **Servlet API** is the contract between that class and the server. It is one of the Java EE specifications, and the one most applications actually use. Versions you will meet:

- Servlet 3.1: 2013, Java EE 7.
- Servlet 4.0: 2017, Java EE 8. The last `javax` version.
- Servlet 5.0: 2020, Jakarta EE 9. Identical to 4.0 except that every package was renamed from `javax` to `jakarta`.
- Servlet 6.0: 2022, Jakarta EE 10.

**JSP**, JavaServer Pages, is HTML with embedded Java that the server compiles into a servlet at first request. It exists because writing HTML as string concatenation inside a servlet was unbearable. Modern applications use a template engine instead and never touch it.

A **WAR file**, Web Application Archive, is a zip file with a fixed layout; the terms WAR, WAR file and archive are used interchangeably below. It was introduced by Servlet 2.2 in 1999 and the layout has not changed since. When a developer hands you a Java web application to run, this file is what they hand you. This is the directory layout as stated by the standard:

```
app.war
├── index.html                  the landing page, static content
├── css/
│   └── style.css               stylesheet, static content
├── js/
│   └── app.js                  browser-side JavaScript, static content
├── images/
│   └── logo.png                image, static content
├── META-INF/                   never served to clients
│   ├── MANIFEST.MF             jar manifest, build metadata
│   └── context.xml             Tomcat-only: per-application context settings
└── WEB-INF/                    never served to clients
    ├── web.xml                 deployment descriptor, see below
    ├── classes/                the application's own compiled classes and resource files
    │   ├── com/
    │   │   └── example/
    │   │       └── app/
    │   │           ├── ApiServlet.class
    │   │           └── LoginServlet.class
    │   └── application.properties
    └── lib/                    third-party jars the application depends on
        ├── commons-lang3-3.14.0.jar
        ├── jackson-databind-2.17.0.jar
        └── mariadb-java-client-3.3.3.jar
```

`web.xml` is the deployment descriptor: the one XML file that tells the server what is inside the archive and how to wire it. It declares:

- servlets: a name for each servlet and the class that implements it. The name is a label chosen by the developer and only used inside `web.xml`; the class is the compiled code under `WEB-INF/classes/`. For example, the name `api` for the class `com.example.app.ApiServlet`
- servlet mappings: which URL pattern matches go to which servlet name. For example, the pattern `/api/*` maps to the name `api`, which the servlet declaration above ties to the class `com.example.app.ApiServlet`; so a request for `/api/users/42` runs `ApiServlet`. Likewise `/login` maps to the name `login`, declared as `com.example.app.LoginServlet`
- filters and filter mappings: classes that run before or after a servlet on a URL pattern, for logging, authentication, compression
- listeners: classes to call when the application starts, stops, or a session is created or destroyed
- session configuration: timeout in minutes, cookie name, cookie flags
- welcome files: what to serve when a request names a directory rather than a file. The application declares one ordered list of filenames: `index.html`, by default, followed by `index.jsp`. Whichever directory is requested, the server looks for those names in that directory, in that order, and serves the first one it finds. Same behaviour as Apache's `DirectoryIndex`
- error pages: which page to show for a given HTTP status or exception
- context parameters: name and value pairs the application reads at startup, the place a developer puts settings they expect you to edit
- security constraints: which URL patterns require authentication and which roles

The server refuses to serve anything under `WEB-INF/` or `META-INF/` as a file: a request for `/WEB-INF/web.xml` or `/META-INF/MANIFEST.MF` gets a 404 by specification, so configuration and code cannot be downloaded by clients even though they sit inside the same archive as the static files. Everything else in the archive is fetchable: `curl http://host/app/js/app.js` returns the file. A client cannot execute a class by asking for it. Classes never map to URLs. `web.xml` (or annotations in the classes, more on those below) declares servlets by name and assigns each one URL patterns. For example, `/api/*` to `ApiServlet` and `/login` to `LoginServlet`. When a request arrives, the server matches its path against those patterns and calls the one servlet that matches; a path that matches nothing gets a 404. A class under `WEB-INF/classes/` that no mapping names is unreachable from outside. This is the opposite of CGI or PHP, where a file on disk under the document root is executable because it is there. `unzip -l app.war` shows you what you were given, and `unzip -p app.war WEB-INF/web.xml` shows what it expects of the server. Since Servlet 3.0 (2009) `web.xml` may be nearly empty, because everything it declares can be declared in the source code, as **annotations**. An annotation is a marker prefixed with `@` written directly above a class in the Java source. The compiler keeps it inside the `.class` file, and the server reads it at startup. So `@WebServlet("/api/*")` above the class `ApiServlet` does the job of both a servlet entry and a servlet-mapping entry in `web.xml`: it names the servlet, ties it to that class and assigns it the URL pattern, in one line next to the code it describes. `@WebFilter` and `@WebListener` do the same for filters and listeners. Developers prefer this because the declaration lives with the class it applies to instead of in a separate file that drifts out of date.

To find the annotations, the server scans every class under `WEB-INF/classes/` and every jar under `WEB-INF/lib/` when the application starts, and combines the annotations it finds with the declarations in `web.xml`. The result is a union, not a concatenation: each servlet, filter and listener is one entry keyed by its name, and when `web.xml` and an annotation both declare the same one, `web.xml` wins. Order does not matter for servlets, since URL matching is by pattern rather than by declaration order; it does matter for filters, which run in the order `web.xml` lists them, and any filters declared only by annotation run after all of those, in an order the specification leaves undefined. A `web.xml` that sets `metadata-complete="true"` tells the server to skip the scan entirely and trust the file alone. Two consequences for you: an empty descriptor does not mean an empty application, so `web.xml` alone no longer tells you which URL patterns the application answers to and which class handles each (its "routes", in the vocabulary of most other web frameworks); the scan is part of why a Java application takes seconds to minutes to start, depending on the application.

## Tomcat

Tomcat exists because Sun needed a reference implementation for the Servlet and JSP specifications. James Duncan Davidson wrote it at Sun in the late 1990s, and Sun donated it to Apache in 1999, where it became Jakarta Tomcat 3.0 implementing Servlet 2.2 and JSP 1.1. A reference implementation is the executable definition of a specification. Performance and operability were not goals, and Tomcat was never redesigned as a product afterwards. Every complaint a sysadmin has about it follows from that.

### What the tarball contains

Unpack `apache-tomcat-9.0.x.tar.gz` and you get one tree:

```
apache-tomcat-9.0.x/
├── bin/
│   ├── catalina.sh             the real start script; startup.sh and shutdown.sh call it
│   ├── startup.sh
│   ├── shutdown.sh
│   └── setenv.sh               does not exist until you create it; where JVM flags go
├── conf/
│   ├── server.xml              the server configuration: ports, TLS, thread pools, virtual hosts
│   ├── context.xml             default per-application settings (sessions, resources), applied to every deployed application
│   ├── web.xml                 server-wide defaults that every application's own web.xml can override
│   ├── catalina.properties     read once at JVM startup: where Tomcat looks for its own jars, and switches such as which jars to skip when scanning
│   ├── logging.properties      what gets logged where
│   ├── tomcat-users.xml        usernames, passwords and roles for the manager application (more on that below)
│   └── Catalina/localhost/     one optional <app>.xml per deployed application: settings for that application only (database pool, context path, session store) kept outside the WAR so a redeploy does not overwrite them
├── lib/                        Tomcat's own jars, and the place people wrongly drop JDBC drivers
├── logs/                       catalina.out and the per-day access and application logs
├── temp/                       java.io.tmpdir for the JVM; private so that the application's scratch files are not in a world-writable /tmp and are removed with the instance
├── webapps/                    drop a WAR here and it deploys; it is unpacked into a directory of the same name next to it
│   ├── ROOT/                   what answers at /
│   ├── manager/                the web management application, enabled by default
│   ├── host-manager/           same idea for virtual hosts: add and remove Host entries at runtime, enabled by default
│   ├── docs/                   the Tomcat documentation as a web application
│   └── examples/               sample servlets and JSPs; delete it, it has had its own CVEs
└── work/                       per-application scratch: JSPs compiled to servlets, and sessions saved across a restart. Clearing it (Tomcat stopped) is the standard fix when a JSP change is not showing or a saved session file from an old build refuses to load
```

Program, configuration, applications, logs and scratch space are all children of one directory. That is the root of the operational trouble: an upgrade means unpacking the new tarball next to the old one and copying `conf/`, `webapps/` and `setenv.sh` across by hand, then diffing `server.xml` against the new default to see what changed.

Tomcat does have the same split Jetty has, under the names `CATALINA_HOME` (the program) and `CATALINA_BASE` (the instance). Set both, give the base its own `conf/`, `webapps/`, `logs/`, `temp/` and `work/`, and the program directory stays clean. Almost nobody does this by hand because the documentation mentions it in passing and every tutorial assumes the one-tree layout. The distro packages do it for you and each does it differently: on Debian the program is `/usr/share/tomcat9`, the instance is `/var/lib/tomcat9`, and `conf/` is a symlink to `/etc/tomcat9`; on EL the paths are `/usr/share/tomcat` and `/var/lib/tomcat` with `/etc/tomcat`. So the first question on any Tomcat box is "tarball or package", because it decides where every file is.

### server.xml

This is the file you will spend your time in. A stripped-down but real one:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               maxThreads="200" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs" prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

Read it as a tree of Java objects, because that is what it is. `Server` is the process; its `port="8005"` is a socket on localhost that stops the server when it receives the string `SHUTDOWN`. `Service` groups one engine with its connectors. `Connector` is a listening socket: port, protocol, thread pool size. `Engine` is the request router; `defaultHost` says which host answers when the request's `Host:` header matches nothing. `Host` is a virtual host, and `appBase="webapps"` is why dropping a WAR there works. `Valve` is a filter that runs on every request through that host, here an access log; `className` is the Java class to instantiate, and every attribute becomes a setter call on it.

That last point is the whole design in one sentence. To change access log format you edit a `pattern` attribute on a `Valve` element, because `AccessLogValve` has a `setPattern` method. To add TLS you add a second `Connector` with `SSLEnabled="true"` and a nested `SSLHostConfig` element, because that is the object graph. The Tomcat documentation for `server.xml` is therefore a list of classes and their attributes, not a list of things you might want to do.

### The other configuration files

`conf/web.xml` is a default deployment descriptor that every application's own `web.xml` is layered on top of. It is where the default servlet (the one that serves static files) and the JSP servlet are declared, and where the global session timeout of 30 minutes lives. If an application does not set its own timeout, this is where it comes from.

`conf/context.xml` holds settings applied to every deployed application: session persistence, resource definitions such as JNDI database pools, cookie settings. A per-application override goes in `conf/Catalina/localhost/<app>.xml` or inside the WAR at `META-INF/context.xml`, and the precedence between those three is where people lose afternoons. The path is the object tree again: `Catalina` is the `Engine` name from `server.xml`, `localhost` is the `Host` name, and there is one file per context under it. It says `localhost` rather than your FQDN because Tomcat's `Host` is a virtual host keyed by the HTTP `Host:` header, and Apache named the default one `localhost` so the tarball works on any machine unedited. Add `<Host name="app.example.org">` to `server.xml` and the matching directory becomes `conf/Catalina/app.example.org/`.

`conf/catalina.properties` sets the class loader search paths and a handful of global switches. You touch it once, to turn off jar scanning for jars you know contain no annotations, because that scan is most of Tomcat's startup time.

`bin/setenv.sh` is where heap size, garbage collector and system properties belong, as `CATALINA_OPTS="-Xmx2g -Duser.timezone=UTC"`. The file does not exist in the tarball and `catalina.sh` sources it only if present, so on a fresh install the JVM runs with default settings and nobody notices until it is slow. The distro packages replace this with `/etc/default/tomcat9` or `/etc/sysconfig/tomcat`, so again the location depends on who packaged it.

### Deployment

Drop `app.war` into `webapps/`. The host's autodeployer polls the directory every few seconds, sees the file, unpacks it into `webapps/app/`, reads `WEB-INF/web.xml`, scans for annotations and starts the application at `/app`. The context path is the filename; `ROOT.war` is the magic name for `/`. There is no command that says "deploy this" and no exit status that says "deploy failed"; the only signal is a stack trace in `catalina.out` and a 404 where the application should be.

Undeploying means deleting `webapps/app.war` and `webapps/app/`. The autodeployer notices and stops the application, but the JVM keeps every class it loaded, every file handle it opened and every thread the application started. Threads, JDBC drivers registered with the runtime, timers and thread-local variables survive as a leak the class loader cannot collect. Deploy the same WAR again and it runs alongside the ghost of the previous one. Tomcat ships a `JreMemoryLeakPreventionListener` to mitigate the known cases, which is an admission. The practice everyone converged on is one WAR per Tomcat, restart the JVM to deploy, which makes the autodeployer and the manager application dead weight you carry anyway.

### The manager application

`webapps/manager/` is a web application that ships with Tomcat and manages Tomcat: a page listing every deployed application with Start, Stop, Reload and Undeploy buttons, an upload form to deploy a WAR from the browser, and a status page showing thread pools and memory. `webapps/host-manager/` does the same for virtual hosts. Both are enabled in the tarball, both are gated only by `conf/tomcat-users.xml`, which ships with every user commented out, and both are reachable on the same port as your application. For a decade "Tomcat manager with default credentials" was a standard entry in penetration test reports, because the moment someone uncommented a user to try it out, `/manager/html` became a remote WAR upload for anyone who guessed the password. The manager also has a text interface at `/manager/text` that build tools and deployment plugins use to push WARs, which is the only reason it survives. If nothing pushes to it, delete `webapps/manager/` and `webapps/host-manager/` on day one; the distro packages ship them as separate optional packages for this reason.

### Logs

`logs/catalina.out` is stdout and stderr of the JVM, appended forever, never rotated by Tomcat itself. Startup messages, stack traces from failed deployments and anything the application prints all land here. `logs/catalina.<date>.log` and `logs/localhost.<date>.log` are Tomcat's own logging, rotated daily by `logging.properties`. `logs/localhost_access_log.<date>.txt` is the access log from the valve above. The distro packages send `catalina.out` to journald or `/var/log/tomcat9/`. When something is wrong, `catalina.out` is where the answer is, and on a busy server it is gigabytes.

### Why it is heavy

Tomcat is not heavy because of the engine. It is heavy because it starts from everything: manager, host-manager, docs and examples applications, a JSP compiler, an autodeployer polling `webapps/`, JMX registered for every component, a default pool of 200 threads. Strip it to one connector and one WAR, delete everything in `webapps/` but your application, and it is 15 MB of jars starting in under a second. The difference from Jetty is direction: Tomcat starts from everything and you remove; Jetty starts from nothing and you add.

### Tomcat 8, 9, 10, 10.1, 11: which one and why

Each Tomcat major version implements exactly one Servlet specification version, and that is the whole difference between them. The engine, the configuration files and the layout barely change; what changes is which API the WAR must have been compiled against.

| Tomcat | Servlet | Package prefix | Platform | Minimum JDK | Status |
|---|---|---|---|---|---|
| 8.5 | 3.1 | `javax` | Java EE 7 | 7 | End of life March 2024 |
| 9.0 | 4.0 | `javax` | Java EE 8 | 8 | Maintained; the last `javax` Tomcat |
| 10.0 | 5.0 | `jakarta` | Jakarta EE 9 | 8 | End of life October 2022 |
| 10.1 | 6.0 | `jakarta` | Jakarta EE 10 | 11 | Maintained |
| 11.0 | 6.1 | `jakarta` | Jakarta EE 11 | 17 | Maintained |

The line that matters is between 9 and 10: 9 is `javax`, everything from 10 on is `jakarta`. A WAR compiled against `javax.servlet` deploys on Tomcat 9 and fails on Tomcat 10 with `ClassNotFoundException: javax.servlet.http.HttpServlet`, because that class no longer exists in the server. A WAR compiled against `jakarta.servlet` fails the same way on 9. Nothing else about the WAR needs to change; the rename is the entire incompatibility. Tomcat 10 ships a migration tool that rewrites the package names inside an old WAR's class files at deploy time, which works for simple applications and not for the ones that matter.

So the rule is: look at the WAR, not the calendar. `unzip -l app.war` and check whether `WEB-INF/lib/` contains jars named `javax.servlet-api` or `jakarta.servlet-api`, or ask the developer which Spring Boot generation it is: Spring Boot 2 is `javax` and needs Tomcat 9, Spring Boot 3 is `jakarta` and needs 10.1 or 11. Tomcat 9 is not old for being the lower number; it is the current server for the entire `javax` world, receives the same security fixes as 11, and will for years, because that world is larger than the `jakarta` one.

10.0 existed for eighteen months as the rename release with no new features and is dead; treat any mention of it as 10.1. 8.5 is end of life and a `javax` application on it moves to 9 with no changes.

This is the situation Jetty 12 was designed to escape: one Jetty serves both `javax` and `jakarta` WARs from one installation, with `ee8` and `ee10` as modules instead of as separate products.

## Jetty

Greg Wilkins started Jetty in 1995 as an embeddable HTTP server. Servlets arrived later as a feature, so its centre of gravity has always been the server, not the specification. It is now an Eclipse project.

Jetty 9, 10 and 11 each targeted one Servlet version, like Tomcat still does. Jetty 12 changed that. The core server knows HTTP, TLS, handlers and deployment and contains no servlet API at all. Each Jakarta EE level is an **environment** you enable: `ee8` (javax, Servlet 4), `ee9` (jakarta, Servlet 5), `ee10` (jakarta, Servlet 6), `ee11` in Jetty 12.1. Several can run in one server at once. This is the `javax` to `jakarta` seam made visible and manageable.

### Home and base

`$JETTY_HOME` is the unpacked distribution. Read-only, never edited, replaced whole at upgrade. Think `/usr/lib/jetty`. Unpack `jetty-home-12.0.x.tar.gz` and you get:

```
jetty-home-12.0.x/
├── start.jar                   the launcher; the only thing you ever run
├── lib/                        every jar Jetty could load, organised by module
│   ├── jetty-server-12.0.x.jar
│   ├── jetty-ee8-servlet-12.0.x.jar
│   └── ...
├── modules/                    one .mod file per feature
│   ├── http.mod
│   ├── https.mod
│   ├── ssl.mod
│   ├── ee8-deploy.mod
│   ├── requestlog.mod
│   └── ...
├── etc/                        XML fragments the modules reference
│   ├── jetty.xml
│   ├── jetty-http.xml
│   └── ...
└── bin/
    └── jetty.sh                init-style start script, optional
```

Note what is missing: no `webapps/`, no `logs/`, no `start.ini`. The distribution cannot run on its own, on purpose. Everything that varies per server lives in a base.

`$JETTY_BASE` is a directory per server instance holding only what differs from home. Think `/etc/jetty/<instance>`. You create it empty and ask `start.jar` to populate it:

```
mkdir -p /opt/jetty/base/app
cd /opt/jetty/base/app
java -jar /opt/jetty/12.0.x/start.jar --add-modules=http,ee8-deploy
```

`start.jar` reads `modules/http.mod` and `modules/ee8-deploy.mod` from home, follows their dependencies (`server`, `ee8-webapp`, `ee8-security`, `deploy` and so on) and writes the result into the base:

```
/opt/jetty/base/app/
├── start.d/
│   ├── http.ini
│   └── ee8-deploy.ini
├── webapps/
└── resources/
```

Nothing was copied from home. The base contains two small ini files and an empty `webapps/` directory. Start the server from inside the base:

```
cd /opt/jetty/base/app
java -jar /opt/jetty/12.0.x/start.jar
```

`start.jar` takes the base from the working directory, or from `-Djetty.base=/opt/jetty/base/app` if you run it from elsewhere. One home can back any number of bases, each on its own port with its own modules. Upgrading Jetty means unpacking a new home, changing one path in whatever starts the server, and starting it; the base is untouched. A diff of the base against an empty directory is exactly your configuration.

### Modules and start.d

A **module** is a file `$JETTY_HOME/modules/<name>.mod` that declares, in named sections, which jars go on the classpath, which XML files to load, which properties exist with what defaults, and which other modules it depends on. `http.mod`, shortened:

```
[description]
Enables a clear-text HTTP connector.

[tags]
connector
http

[depend]
server

[xml]
etc/jetty-http.xml

[ini-template]
## Connector host/address to bind to
# jetty.http.host=0.0.0.0

## Connector port to listen on
# jetty.http.port=8080

## Connector idle timeout in milliseconds
# jetty.http.idleTimeout=30000
```

Enabling the module copies the `[ini-template]` section into `$JETTY_BASE/start.d/http.ini` with one line added at the top:

```
--module=http
# jetty.http.host=0.0.0.0
# jetty.http.port=8080
# jetty.http.idleTimeout=30000
```

To change the port, uncomment the line and edit it. That is the entire configuration model: one ini file per concern, containing only the properties that concern has, with the defaults visible as comments. There is no XML for you to edit; `etc/jetty-http.xml` in home reads `jetty.http.port` and you never open it.

JVM flags go in the same files, as lines starting with `--exec` and `-X`. A `start.d/jvm.ini` containing:

```
--exec
-Xmx2g
-Duser.timezone=UTC
```

makes `start.jar` fork a second JVM with those flags and run the server in it. Heap size sits in the base next to the port, not in a shell script somewhere else.

Two commands tell you what a base will do before you start it:

```
java -jar /opt/jetty/12.0.x/start.jar --list-modules=enabled
java -jar /opt/jetty/12.0.x/start.jar --list-config
```

The first lists the enabled modules and, for each, which ini file enabled it and what it depends on. The second prints the resolved result: every property with its final value and where it came from, the full classpath in order, and every XML file that will be loaded in order. If a value is not what you expect, this output says which file set it. Tomcat has no equivalent; the effective configuration of a running Tomcat exists only inside the JVM.

### Deployment

The `ee8-deploy` module (or `ee9-deploy`, `ee10-deploy`) does two things. It puts the servlet API jars for that environment on a class loader of their own, and it starts a scanner on `$JETTY_BASE/webapps/`. Drop `app.war` there and the scanner deploys it at `/app`; `ROOT.war` deploys at `/`. Same convention as Tomcat, with one addition: when several environments are enabled in one base, a WAR needs a sidecar telling the scanner which environment it belongs to, `app.properties` next to `app.war` containing `environment=ee8`. With one environment enabled, it is the default and the sidecar is unnecessary.

The WAR is unpacked into `$JETTY_BASE/work/` if that directory exists, otherwise into a temporary directory that is deleted on exit. Create `work/` (the `work` module does it) if you want the unpacked tree to survive restarts and be inspectable. Neither path is anything the application knows about; the application's own configuration and data live wherever the application says, and Jetty does not read them.

A failed deployment looks like this on the console: a `WARN` line naming the WAR and the exception, then the stack trace, then `Started Server` anyway with the rest of the server up and the failed context returning 503. Jetty does not refuse to start because one WAR is broken. `start.jar` exits non-zero only if the server itself cannot start, a port in use for instance.

### Logs

By default Jetty logs to stderr, which under systemd is the journal. The `requestlog` module adds an NCSA-format access log under `$JETTY_BASE/logs/`, rotated daily; the `logging-logback` and `logging-log4j2` modules route the server's own log through those frameworks if you want files and levels. Nothing is appended forever by default, because there is no `catalina.out`.

### Running it as a service

A base is a working directory and a command line, so the systemd unit is short:

```
[Unit]
Description=Jetty app
After=network.target

[Service]
User=jetty
WorkingDirectory=/opt/jetty/base/app
ExecStart=/usr/bin/java -jar /opt/jetty/12.0.x/start.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

One unit per base; two bases on one machine are two units with two working directories and two ports, sharing one home. The `jetty.sh` script in `bin/` is for systems without systemd and you do not need it.

## The four directories

For any Java web application on Jetty there are four places, and confusion comes from collapsing them:

1. The Jetty program: `$JETTY_HOME`, here `/opt/jetty/12.0.x`. Never edited. Replaced whole at upgrade.
2. The Jetty instance config: `$JETTY_BASE`, here `/opt/jetty/base/app`. Port, environment, modules, JVM flags, the WAR. The only thing you edit.
3. The application's own config: wherever the application reads it from, typically `/etc/<app>/`. Database connection, data paths, external services. Jetty does not read this; the application does, after Jetty has started it.
4. The application's data: typically a `/data` disk. Jetty never touches it.

Jetty starts, reads 2, loads the WAR from 2, the WAR starts, reads 3, works on 4.

On Tomcat the same four exist, but 1 and 2 are usually one tree, which is why people cannot tell which of their edits are configuration and which are the program.

## Translation table

| Java word | What you would call it |
|---|---|
| JVM | the runtime process, the thing in `ps` |
| JDK | the runtime plus compiler, the package you install |
| bytecode | the portable machine code the JVM executes |
| heap | the memory pool for objects, `-Xmx` sets the ceiling |
| GC | the garbage collector, the JVM's memory reclaimer |
| JIT | the runtime compiler that makes it fast after warm-up |
| Java SE | the runtime and standard library |
| Java EE, Jakarta EE | the server-side spec bundle |
| servlet | a request handler class |
| servlet container | the application server |
| JSP | a server-side template |
| WAR | the application package |
| web.xml | the application's deployment descriptor |
| context | a deployed application and its URL prefix |
| home | the installed program |
| base | the instance config directory |
| module | a plugin definition plus its ini |
| environment | a Servlet API version (ee8, ee9, ee10) |
| bean | a class with getters and setters |
| record | an immutable struct, Java 16 onward |
| JMX | the JVM's management interface |
| JDBC | the database driver API |
| classpath | the library search path |

## Other words you will meet

**Bean**: a Java class with a no-argument constructor and `getX()` and `setX()` pairs. Sun coined it in 1996 for GUI components, then reused the word for everything: Enterprise JavaBeans for remoting, MBeans for management, Spring beans for dependency injection. Same convention, unrelated things.

**Record**: the modern replacement for the data-carrier half of the bean. Standard since Java 16 (2021). The bean version of a user object:

```java
public class User {
    private final String name;
    private final int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    // equals, hashCode, toString: thirty more lines
}
```

The record version:

```java
public record User(String name, int age) {}
```

Constructor, accessors, `equals`, `hashCode` and `toString` are generated by the compiler. Operationally it means nothing, but if you read stack traces or configuration classes from a modern application you will see it, and it tells you the code base is JDK 17 or later.

**EJB**, Enterprise JavaBeans: Java EE's remoting component model. It ran over RMI/IIOP, which is CORBA's wire protocol, so an EJB was a Java-only CORBA object with container-managed transactions wrapped around it. Same distributed-object idea, same failure modes, dead for the same reasons. You will see the acronym in old documentation and nowhere else.

**JMX**, Java Management Extensions: the JVM's built-in management interface. Components register MBeans with readable attributes and callable operations; jconsole, VisualVM or a Prometheus exporter connects over RMI to read thread counts, session counts or trigger a garbage collection. Useful in a monitoring stack, dead weight in a lab.

**Spring, Spring Boot**: not Java EE. A framework that grew as a reaction to Java EE's weight, using plain classes with dependency injection instead of EJBs. Spring Boot bundles a servlet container inside the application jar so you do not need Tomcat or Jetty at all. When a Spring Boot application is instead built as a plain WAR, it runs on either.

## Where to read next

The Jetty 12 Operations Guide at jetty.org is the only current book on running Jetty; there is no printed one. The Eclipse Foundation article "Jakarta EE 8: Past, Present, and Future" covers the platform history from 1996. The Register's March 2018 article on the Jakarta rename covers the trademark dispute. Nothing covers Tomcat from the operator's side; this document is the closest thing.
