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

To find the annotations, the server scans every class under `WEB-INF/classes/` and every jar under `WEB-INF/lib/` when the application starts, and combines the annotations it finds with the declarations in `web.xml`. The result is a union, not a concatenation: each servlet, filter and listener is one entry keyed by its name, and when `web.xml` and an annotation both declare the same one, `web.xml` wins. Order does not matter for servlets, since URL matching is by pattern rather than by declaration order; it does matter for filters, which run in the order `web.xml` lists them, with annotated filters appended after. A `web.xml` that sets `metadata-complete="true"` tells the server to skip the scan entirely and trust the file alone. Two consequences for you: an empty descriptor does not mean an empty application, so `web.xml` alone no longer tells you the routes; and the scan is part of why a Java application takes seconds to start.

## Tomcat

Tomcat exists because Sun needed a reference implementation for the Servlet and JSP specifications. James Duncan Davidson wrote it at Sun in the late 1990s, and Sun donated it to Apache in 1999, where it became Jakarta Tomcat 3.0 implementing Servlet 2.2 and JSP 1.1. A reference implementation is the executable definition of a specification. Performance and operability were not goals, and Tomcat was never redesigned as a product afterwards. Every complaint a sysadmin has about it follows from that.

Configuration is split across `server.xml`, `context.xml`, `web.xml`, `catalina.properties`, `logging.properties` and `setenv.sh`, with the same setting sometimes valid in three of them and the precedence undocumented. `server.xml` is literally Tomcat's internal object tree serialised to XML: Server contains Service contains Engine contains Host contains Context, with Connectors and Valves hung off the nodes. You are not saying "listen on 8080 with TLS"; you are instantiating Java classes by name and setting their properties by attribute.

The distribution and the instance are one tree. The tarball puts `conf/`, `webapps/`, `logs/` and `work/` inside the program directory, so an upgrade means unpacking a new tarball and diffing your edits back in. `CATALINA_HOME` versus `CATALINA_BASE` exists to separate them, but the packages, the documentation and every tutorial ignore it. Debian and Red Hat each fight the layout differently with symlinks, so where a file lives depends on who packaged it.

Deployment is convention rather than declaration. Drop a WAR in `webapps/`, the autodeployer explodes it, the context path comes from the filename, `ROOT.war` is a magic name. Undeploying means deleting the directory and hoping, because the JVM keeps class loaders and file handles on what it exploded, and anything the application started itself (threads, JDBC drivers, timers) survives as a leak. Tomcat ships a memory-leak-protection listener, which is an admission. The working practice everyone converged on is one WAR per Tomcat and a JVM restart to deploy, which makes the autodeployer and the manager application dead weight you carry anyway.

Tomcat is not heavy because of the engine. It is heavy because it starts from everything: manager, host-manager, docs and examples applications, a JSP compiler, an autodeployer polling `webapps/`, JMX registered for every component, a default pool of 200 threads. Strip it to one connector and one WAR and it is small. The difference from Jetty is direction: Tomcat starts from everything and you remove; Jetty starts from nothing and you add.

## Jetty

Greg Wilkins started Jetty in 1995 as an embeddable HTTP server. Servlets arrived later as a feature, so its centre of gravity has always been the server, not the specification. It is now an Eclipse project.

Jetty 9, 10 and 11 each targeted one Servlet version, like Tomcat still does. Jetty 12 changed that. The core server knows HTTP, TLS, handlers and deployment and contains no servlet API at all. Each Jakarta EE level is an **environment** you enable: `ee8` (javax, Servlet 4), `ee9` (jakarta, Servlet 5), `ee10` (jakarta, Servlet 6), `ee11` in Jetty 12.1. Several can run in one server at once. This is the `javax` to `jakarta` seam made visible and manageable.

### Home and base

`$JETTY_HOME` is the unpacked distribution. Read-only, never edited, replaced whole at upgrade. Think `/usr/lib/jetty`.

`$JETTY_BASE` is a directory per server instance holding only what differs from home: which modules are enabled, their property values, any XML overrides and the WARs. Think `/etc/jetty/<instance>`.

You start the program and tell it which config directory to use: `java -jar $JETTY_HOME/start.jar` run from inside the base. One home can back several bases. Upgrading Jetty is unpacking a new home and pointing the base at it; nothing in the base changes. A diff of your base is exactly your configuration. Jetty 12 ships a home with no `webapps/` in it, so running from the distribution directory no longer works even by accident.

### Modules and start.d

A **module** is a file `$JETTY_HOME/modules/<name>.mod` that declares jars, XML, properties and dependencies. `http`, `https`, `ssl`, `ee8-deploy`, `gzip`, `requestlog` are modules. Enabling one writes `$JETTY_BASE/start.d/<name>.ini` containing the module name and its properties, commented out with defaults shown. You edit the ini to set the port. At startup `start.jar` resolves the module graph, builds the classpath and the XML list, and can print the result with `--list-config`. There is one place per concern and a command that shows the effective configuration. That is the whole answer to the Tomcat configuration problem.

JVM flags belong in the same ini files, as `--exec` lines, so heap and GC settings live next to the module they serve rather than in a shell script that does not exist until you create it.

### Deployment

Enabling `ee8-deploy` adds the ee8 servlet jars in their own class loader and a deployer that scans `$JETTY_BASE/webapps/` for WARs. The WAR is unpacked into `$JETTY_BASE/work/` if that directory exists, otherwise into a temp directory. Neither path is anything the application knows about; the application's own configuration and data live wherever the application says, and Jetty does not read them.

## The four directories

For any Java web application on Jetty there are four places, and confusion comes from collapsing them:

1. The Jetty program: `$JETTY_HOME`. Never edited.
2. The Jetty instance config: `$JETTY_BASE`. Port, environment, modules, the WAR.
3. The application's own config: wherever the application reads it from, typically `/etc/<app>/`. Database, data paths, external services. Jetty does not read this; the application does.
4. The application's data: typically a `/data` disk.

Jetty starts, reads 2, loads the WAR from 2, the WAR starts, reads 3, works on 4.

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
