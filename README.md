# MS SQL in Python on MacOS
_Targeting OS X 10.11+ and Python 2.7 without using Homebrew, MacPorts, or Docker_

By Neil C. Obremski

  * Written circa November 2015: tested on OS X 10.11.1 "El Capitan".
  * Updated September 2017: tested on OS X 10.12.6 "Sierra".
  * Updated November 2019: tested on MacOS 10.15 "Catalina" and put on GitHub.

## Step 1. Xcode:
Install Xcode from the App Store. This beast is over 4gb and can take a couple of hours to download and install. Start doing it now!

After it’s finished, open a Terminal and install command-line tools with the following:

```sh
xcode-select --install
```

You’ll need Xcode and the command-line tools to configure and make the various libraries from their tarballs.

## Step 2. unixODBC:
MacOS _used_ to come with iODBC¹ as its default ODBC driver manager but now it’s a clean slate. And since new versions of Pyodbc are unfortunately hard-coded to compile against unixODBC, that’s the driver manager you’ll want to install to avoid headaches.

Run the following terminal commands to download the tarball, build, and install unixODBC:

```sh
cd ~/Downloads
curl -O ftp://ftp.unixodbc.org/pub/unixODBC/unixODBC-2.3.4.tar.gz
tar xzvf unixODBC*.tar.gz
cd unixODBC*
./configure
make
sudo make install
```

This will install files under `/usr/local/` like `/usr/local/include/sql.h` which FreeTDS and Pyodbc² need in order to build properly. In particular, `/usr/local/etc/odbcinst.ini` specifies system-wide ODBC drivers which we will configure in step 3, installing FreeTDS.

Run `isql` with no arguments to acid test the unixODBC install and make sure it can load `/usr/local/lib/libodbc.2.dylib`. To really use isql, first FreeTDS must be installed and then DSN sections must be added to `/usr/local/etc/odbc.ini`. For instructions and examples of this, check out [unixODBC - MS SQL Server/PHP](http://www.unixodbc.org/doc/FreeTDS.html) and use [`osql` to troubleshoot](http://www.freetds.org/userguide/odbcdiagnose.htm). However, it’s really only worth doing this if there is trouble with Pyodbc.

## Step 3. FreeTDS:
FreeTDS is the actual _driver_ for connecting to Microsoft SQL Server whereas unixODBC is the ODBC _driver manager_ that loads it. Prior to starting this document, FreeTDS hadn’t been updated in years and had stalled out at version 0.91 in 2011. Suddenly it sprang back to life in May 2015 and the download URL changed, presumably to prevent downloading a buggy version.

```sh
cd ~/Downloads/
curl -O ftp://ftp.freetds.org/pub/freetds/stable/freetds-1.1.tar.gz
```

Here’s how to build the tarball:

```sh
cd ~/Downloads/
tar xzvf freetds*.tar.gz
cd freetds*
./configure --with-unixodbc=/usr/local --with-tdsver=7.2
make
```

Test the build success by running the following and looking for `unixodbc: yes` in the output:

```sh
src/apps/tsql -C
```

You test that FreeTDS can access your database server like so:

```sh
src/apps/tsql -H host -p port -U username
```

If all goes well then you’ll be asked for a password and then left at a simple prompt where you can send SQL statements to the server. After writing a statement, make sure to add `GO` on a separate line or it won’t execute:

```text
locale is "en_US.UTF-8"
locale charset is "UTF-8"
using default charset "UTF-8"
1> SELECT COUNT(*) FROM mytable WITH (NOLOCK)
2> GO

54321
(1 row affected)
1> quit
```

Install the FreeTDS library and tools you built by running:

```sh
sudo make install
```

Finally, add FreeTDS to the unixODBC driver list otherwise it won’t know where to find it. Add the following lines to `/usr/local/etc/odbcinst.ini` (which is probably blank):

```ini
[FreeTDS]
Driver = /usr/local/lib/libtdsodbc.so
Setup = /usr/local/lib/libtdsodbc.so
FileUsage = 1
```

One very important thing to note is the "TDS protocol version": Microsoft SQL Server requires at least 7. And not just that, because this isn’t a number but an exact string: you must specify "7.2" otherwise FreeTDS disregards the value. So if tsql quickly bombs out with the message "_There was a problem connecting to the server_" then you’re probably trying to connect with a lower version like 5.0. One of the many ways to set this is to create `~/.freetds.conf` and add the following lines:

```ini
[global]
  tds version = 7.2
```

Nearly there! Now we just need to be able to do this in Python!

## Step 4: Pyodbc:

Pyodbc stands for Python ODBC and is sometimes written PyODBC because of that. This module has _no direct knowledge_ of Microsoft SQL Server. All it knows is ODBC which is a generic way of communicating with databases. The default version plays favorite to unixODBC in its build configuration.

Make sure you have pip (`sudo easy_install pip`) and then run the following command:

```sh
sudo pip install --global-option="build_ext" --global-option="-L/usr/local/lib" pyodbc
```

Now you can test Pyodbc in python like so:

```python
import pyodbc
db = pyodbc.connect("DRIVER=FreeTDS;SERVER=host;Port=port;UID=username;PWD=password;Database=database")
list(db.execute("SELECT COUNT(*) FROM mytable WITH (NOLOCK)"))
```

Ta da!

## Troubleshooting

I’ve spent countless hours futzing with Pyodbc, Mac OS X from Mavericks to Catalina, and SIP to `/usr/local/lib` so I’m sure I’ve run into the same problem and hopeful I can help!

### ld: library not found for -lodbc

If this occurs when running pip install then it’s probably because the library ("odbc" in this case) is in /usr/local/lib and not `/usr/lib`. Normally you could set `LIBRARY_PATH`, `LD_LIBRARY_PATH`, or `DYLB_LIBRARY_PATH` but I haven’t had any luck with these, possibly due to SIP. Instead, try using PIP’s "global-option" parameter like so:

```sh
pip install --global-option=build_ext --global-option="-L/usr/local/lib" <module>
```

Apple’s SIP (System Integrity Protection), which is present in OS X since El Capitan, prevents unixODBC from being installed into `/usr/lib/` where PIP compilations normally look for library files. The only way around this is to pass `-L/usr/lib/local` into the linker (`ld`) command. In the setup.py for the PIP module (such as pyodbc), add the following to [the compiler settings object](https://github.com/mkleehammer/pyodbc/pull/278):

```python
settings['library_dirs'] = [ '/usr/local/lib' ]
```

It’s interesting to note that if you change this library to “iodbc” on a clean OS X install, it will successfully build because Apple is still including the binary with the operating system. This doesn’t help us, however, because we need the headers to build against which is why we get unixODBC instead of reusing iODBC.

References:

  * StackOverflow: [python pip specify a library directory and an include directory](https://stackoverflow.com/questions/18783390/python-pip-specify-a-library-directory-and-an-include-directory)
  * StackOverflow: [Telling ld where to look for directories via an environment variable](https://stackoverflow.com/questions/2226585/telling-ld-where-to-look-for-directories-via-an-environment-variable)

### Passwords containing []{}(),;?*=!@

Pyodbc passes the connection string as is to the ODBC driver so this has to do with connection strings in general. Enclose the value in curly braces to make it work:

```text
...;PWD={{+Y3mv*S)chzE[1(aQ "};...
```

References:

  * https://msdn.microsoft.com/en-us/library/system.data.odbc.odbcconnection.connectionstring(v=vs.110).aspx
  * https://www.connectionstrings.com/formating-rules-for-connection-strings/
  * https://stackoverflow.com/questions/22398212/escape-semicolon-in-odbc-connection-string-in-app-config-file

### Django

_The following was written in April 2015 and probably needs to be updated._

  1. Clone `https://github.com/neilobremski/django-pyodbc-azure` to `~/django-pyodbc-azure`
  2. `export PYTHONPATH=~/django-pyodbc-azure:$PYTHONPATH`
  3. Modify your `local_settings.py` (see below)

```python
import pyodbc
DATABASES['mssql'] = {
    'ENGINE': 'sql_server.pyodbc',
    'NAME': 'mssql',
    'USER': 'user',
    'PASSWORD': '...',
    'HOST': 'host',
    'PORT': 9001,
    'OPTIONS': {
        'driver_charset': 'latin1',
        'host_is_server': True,
        'driver': '/usr/local/lib/libtdsodbc.so',
        'extra_params': 'TDS_Version=7.2',
    }
}
```

The two things I added to Azure were `driver_charset` and the ability to specify a full path to `libtdsodbc.so`. Azure currently supports the option `driver_requires_utf8`, but it turns the Microsfot SQL Server and/or iODBC is communicating with the latin1 character set.

This prevents the following errors which you may have seen if you ever tried to get this thing working on your Mac and didn't install unixODBC ...

Exceptions caused by driver issues (iODBC requires full path to driver file):

```text
Error: ('00000', '[00000] [iODBC][Driver Manager]dlopen({{FreeTDS}}, 6): image not found (0) (SQLDriverConnect)')
Error: ('00000', '[00000] [iODBC][Driver Manager]dlopen({{/usr/local/Cellar/freetds/0.91_1/lib/libtdsodbc.so}}, 6): image not found (0) (SQLDriverConnect)')
```

Exceptions caused by Unicode (FreeTDS does not support Unicode):

```text
ProgrammingError: ('The SQL contains 0 parameter markers, but 8 parameters were supplied', 'HY000')
ProgrammingError: No results.  Previous SQL was not a query.
```

Exceptions caused by encoding (utf-8 when latin1 is used):

```text
UnicodeDecodeError: 'utf8' codec can't decode byte 0xa0 in position 68: invalid start byte
```

## Links

  * [unixODBC](http://www.unixodbc.org)
  * [Pyodbc](https://github.com/mkleehammer/pyodbc)
  * [Mac ODBC Help](http://www.macsos.com.au/MacODBC/index.html)
  * [INSTALLING & DEBUGGING ODBC ON MAC OS X](http://www.cerebralmastication.com/2013/01/installing-debugging-odbc-on-mac-os-x/)


## Footnotes
¹ OS X 10.12 "Sierra" comes with binaries for iODBC but no headers to compile against.
² `/usr/local/lib/libodbc.dylib` is the OS X shared library "odbc" linked by Pyodbc
