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

Names you will meet: **JDK**, Java Development Kit, is the compiler plus the runtime; **JRE**, Java Runtime Environment, was the runtime alone and is no longer shipped separately, so you install a JDK even to run things, the `-headless` package variants just omit the desktop libraries. **OpenJDK** is the open-source codebase; Oracle, Red Hat, Eclipse Temurin, Amazon Corretto, Azul and Microsoft all build it, pass the same compatibility kit, and differ in support terms and patch cadence, not behaviour. Version numbering went 1.0 to 1.8, then 9 onward with a new release every six months and a long-term-support release every two years: 8, 11, 17, 21, 25. Applications declare the minimum they compile against; Spring Boot 2 era applications want 11 or 17, Jetty 12 wants 17.

Which JDK to install: the distribution package, `java-17-openjdk-headless` on EL, `openjdk-17-jre-headless` on Debian. It is the same OpenJDK, patched by the distro, updated by `dnf` or `apt` with everything else, and inside the distro's security process. A vendor tarball is for two cases only: a version the distribution does not ship, or a container image with a specific JDK baked in. Pick an LTS version and never a six-month release or an early-access build: vendors support only LTS, ship security updates quarterly for those, and reserve the right to ship nothing for anything else. Do not start anything new on 8; it is 2014 code kept alive on paid support.

Why old applications refuse to run on a new JDK: Java 9 introduced the module system and Java 11 removed the Java EE pieces that had been bundled with the runtime since the 2000s (XML binding, SOAP, activation, CORBA). An application compiled against 8 that used them starts on 11 and dies with `ClassNotFoundException: javax.xml.bind.JAXBContext` or similar. That is the whole of the 8 to 11 pain: not the language, the missing jars. Applications fixed it by adding those libraries as ordinary dependencies, and anything maintained since 2019 has done so; anything that has not is telling you something about its maintenance.

Flags go on the `java` command line. `-X` flags are standard across JVMs (`-Xmx`), `-XX:` flags are HotSpot-specific and change between versions (`-XX:+UseG1GC`), `-D` sets system properties the application reads (`-Duser.timezone=UTC`). Where those flags live is a per-server question, and one of the things that separates Tomcat from Jetty below.

## Servlets, JSP and the WAR

A **servlet** is a Java class that the server instantiates once and calls for every HTTP request. The server owns the sockets, the threads and the lifecycle; the class gets a request object and a response object. It replaced CGI, which forked a process per request, and in 1996 forking a JVM per request took seconds. One instance serves all requests concurrently, so a servlet is effectively a singleton and its instance fields are shared state.

The **Servlet API** is the contract between that class and the server. It is one of the Java EE specifications, and the one most applications actually use. Versions you will meet: Servlet 3.1 (2013, Java EE 7), Servlet 4.0 (2017, Java EE 8, the last `javax` version), Servlet 5.0 (Jakarta EE 9, the rename), Servlet 6.0 (Jakarta EE 10).

**JSP**, JavaServer Pages, is HTML with embedded Java that the server compiles into a servlet at first request. It exists because writing HTML as string concatenation inside a servlet was unbearable. Modern applications use a template engine instead and never touch it.

A **WAR**, Web Application Archive, is a zip file with a fixed layout. Servlet 2.2 introduced it in 1999 and it has not changed since. It is the deliverable you were handed.

```
app.war
├── index.html               static content, served as-is
├── css/  js/  images/       more static content
├── META-INF/
│   ├── MANIFEST.MF          jar manifest, build metadata
│   └── context.xml          Tomcat-only: per-application context settings
└── WEB-INF/                 never served to clients
    ├── web.xml              deployment descriptor: servlets, mappings, filters, session config
    ├── classes/             the application's own compiled classes and resource files
    └── lib/                 third-party jars the application depends on
```

The server refuses to serve anything under `WEB-INF/` as a file: a request for `/WEB-INF/web.xml` gets a 404 by specification, so configuration and code cannot be downloaded by clients even though they sit inside the same archive as the static files. Nor can a client execute a class by asking for it. Classes never map to URLs. `web.xml` (or annotations in the classes) declares servlets by name and assigns each one URL patterns, for example `/api/*` to `ApiServlet` and `/login` to `LoginServlet`. When a request arrives, the server matches its path against those patterns and calls the one servlet that matches; a path that matches nothing gets a 404. A class under `WEB-INF/classes/` that no mapping names is unreachable from outside. This is the opposite of CGI or PHP, where a file on disk under the document root is executable because it is there. Everything outside `WEB-INF/` is a URL. `unzip -l app.war` shows you what you were given, and `unzip -p app.war WEB-INF/web.xml` shows what it expects of the server. Since Servlet 3.0 `web.xml` may be nearly empty, with servlets declared by annotations inside the classes instead, so an empty descriptor does not mean an empty application.

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
