#Meeting minutes of Meeting#11.

# Meeting Note #
1. Debug MySQL.
  * use Eclipse CDT on Linux
  * or use Visual Studio on Windows

2. Modify 1st design.
  * redesign structure "escrow": move to structure "que\_bue"? or remove some unnecessary fields?
  * Explore spinlock supported in MySQL




# Action Points #

1. Debug MySQL in Eclipse.

  * First, run "./configure --prefix=/your/path --with-debug --with-plugins=innobase".
    * **Note: the configure option "--with-innodb" listed in MySQL documentation is not recognizable by MySQL**

  * Create an empty Eclipse C++ project, reference link: [Create A Eclipse Project](http://forge.mysql.com/wiki/Eclipse/CDT_on_Linux_and_Mac_OS_X#Things_I_haven.27t_figured_out_yet)
    * Link the source code with project: "New" -> "Folder" -> "Advanced" -> "Link to folder in the file system" and choose directory and click "Finish" button.
    * In the "Properties" -> "C/C++ Build" view disable "Makefile generation"

  * Build the project
    * Choose "Debug" configuration for project
    * Build all

  * Pre-setup for debugging.
    * Running "scripts/mysqld\_safe --skip-grant-tables &" in a command prompt
    * Running "client/.libs/mysql --user=mysql" in another command prompt
    * Set password for one user(eg. root)
    * Create a table using "ENGINE=INNODB" option and insert some records
    * Stopping "mysqld" server
      * If installed MySQL from binary package starts automatically when Linux starts, stop it via command "sudo /etc/init.d/mysql stop"

  * Running project
    * Create a debug configuration of mysql
    * Start "mysqld" inside Eclipse(with debug mode)
    * Running "mysql" outside of Eclipse to connect to "mysqld" started in previous step
    * Set breakpoints

  * Things haven't figured out
    * The default password of user "mysql" and "root" doesn't work after initialized databse



2. Debug MySQL in Visual Studio.
  * Mainly followed this documentation, [Install from Source Code on Windows](http://dev.mysql.com/doc/refman/5.1/en/windows-source-build.html)
    * **Note: Currently MySQL doesn't support Visual Studio 2010**
      * Create another .bat file under directory PATH/win/build-vs10.bat with follow command
        * cmake -G "Visual Studio 10"
    * Need to install ActivePerl to run test scripts, since cygwin doesn't automatically support perl.

  * Error: "LINK: fatal error LNK1181: cannot open input file Debug\sql.lib"
    * haven't figured out how to solve it

3. Observations of running "UPDATE" statement.
  * 11 threads total under _mysqld_ program
    * basic analyze
      * 2 threads running on common mysqld
      * 8 threads running on innobase storage
      * 1 thread running through common mysqld and innobase
    * detailed thread stack
      * **main()->handle\_connection\_sockets()** _-> select() ->__kernel\_vsyscall()_
      * io\_handler\_thread() -> fil\_aio\_wait() -> os\_aio\_simulated\_handle() -> os\_event\_wait\_low() -> safe\_cond\_wait() _-> pthread\_cond\_wait() ->__kernel\_vsyscall()_
      * io\_handler\_thread() -> fil\_aio\_wait() -> os\_aio\_simulated\_handle() -> os\_event\_wait\_low() -> safe\_cond\_wait() _-> pthread\_cond\_wait() ->__kernel\_vsyscall()_
      * io\_handler\_thread() -> fil\_aio\_wait() -> os\_aio\_simulated\_handle() -> os\_event\_wait\_low() -> safe\_cond\_wait() _-> pthread\_cond\_wait() ->__kernel\_vsyscall()_
      * io\_handler\_thread() -> fil\_aio\_wait() -> os\_aio\_simulated\_handle() -> os\_event\_wait\_low() -> safe\_cond\_wait() _-> pthread\_cond\_wait() ->__kernel\_vsyscall()_
      * **srv\_lock\_timeout\_thread()** -> os\_thread\_sleep() _-> select() ->__kernel\_vsyscall()_
      * **srv\_error\_monitor\_thread()** -> os\_thread\_sleep() _-> select() ->__kernel\_vsyscall()_
      * **srv\_monitor\_thread()** -> os\_thread\_sleep() _-> select ->__kernel\_vsyscall()_
      * **srv\_master\_thread()** -> os\_event\_wait() -> safe\_cond\_wait() _-> pthread\_cond\_wait() ->__kernel\_vsyscall()_
      * **signal\_hand()** _-> sigwait() -> do\_sigwait() ->__kernel\_cond\_wait()_
    * most important thread handling the UPDATE
      * row\_upd\_step() at _row0upd.c_
      * row\_update\_for\_mysql() at _row0mysql.c_
      * ha\_innobase::update\_row() at _ha\_innodb.cc_
      * handler::ha\_update\_row() at _handler.cc_
      * mysql\_update() at _sql\_update.cc_
      * mysql\_execute\_command() at _sql\_parse.cc_
      * mysql\_parse() at _sql\_parse.cc_
      * dispath\_command() at _sql\_parse.cc_
      * do\_command() at _sql\_parse.cc_
      * handle\_one\_connection() at _sql\_connect.c_
      * start\_thread()
      * clone()



3. Redesign.

4. Spinlock in MySQL.