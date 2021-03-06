layout: true
class: inverse, larger

---
class: special, middle
# uWSGI

slides by @natefoo, @Slugger70

.footnote[\#usegalaxy / @galaxyproject]

---
# What is uWSGI?

Web/application/WSGI server

Replaces `Paste#http` (see `galaxy.ini`)

---
class: largeish
# Why uWSGI?

- Python GIL is a severe limitation for multicore servers
- Work around the GIL without manually managing multiple Galaxy processes

![Python GIL](images/gil.png)

.footnote[Image credit: [Dariusz Fryta](http://www.tivix.com/blog/lets-go-python/)]

---
# Why uWSGI?

- Isolate job functions from web functions
  - Can be done with Paste, but uWSGI does it better/simpler
- Built in load balancing
- Speak native high performance uWSGI protocol to nginx
- Uninterrupted restarting

---
# Other features

Can do anything you can imagine: [uWSGI configuration options](http://uwsgi-docs.readthedocs.io/en/latest/Options.html)

---
class: largeish
# What's changed?

Galaxy < 18.01: uWSGI possible and documented but not provided

Galaxy >= 18.01:

- uWSGI is the default server for new installs
- YAML is the default config format for new installs
- Lots of development work into improving Galaxy/uWSGI integration
- Job handlers run as uWSGI *Mules*
- uWSGI now comes with Galaxy (as a *wheel*)<sup>[1]</sup>

.footnote[
<sup>[1] Eagle-eyed viewers might have noticed that it was added to Galaxy in 17.09, but it has not been used by Galaxy itself until 18.01.</sup>
]

---
class: middle, top-title

## uWSGI

- **Configuration**
- uWSGI wheel
- Job handler mules

---
# uWSGI as the default

If you have an existing Galaxy server using `galaxy.ini` and `run.sh`, **nothing changes**

Only new installs use uWSGI by default

---
class: largeish

## uWSGI as the default

**Previously**, a config file was *required*.

We hid this detail a bit by reading `galaxy.ini.sample` if `galaxy.ini` was not present

--

**Now**:
- If no config exists, all necessary defaults are generated as command line arguments to uWSGI
- If a config exists, it is parsed, and any missing required options are passed as command line arguments to uWSGI
- The sample is never used

---

## uWSGI default command line

Without a config file, the uWSGI command line is:

```sh-session
$ /home/nate/galaxy/.venv/bin/python2.7 .venv/bin/uwsgi \
    --module 'galaxy.webapps.galaxy.buildapp:uwsgi_app()' \
    --virtualenv /home/nate/galaxy/.venv --pythonpath lib \
    --threads 4 --http localhost:8080 \
    --static-map /static/style=/home/nate/galaxy/static/style/blue \
    --static-map /static=/home/nate/galaxy/static --die-on-term \
    --hook-master-start 'unix_signal:2 gracefully_kill_them_all' \
    --hook-master-start 'unix_signal:15 gracefully_kill_them_all' \
    --enable-threads --py-call-osafterfork
```

---
class: middle, top-title

## uWSGI command line

Behind the scenes, `run.sh` calls `scripts/get_uwsgi_args.py`, which:
- locates your config file (if any),
- parses it to determine whether you have set any of the default (or conflicting) options,
- determines the flags needed to run Galaxy

You can take "full control" over this process by skipping `run.sh` and simply
calling `uwsgi` directly.

---
# uWSGI communication

- uWSGI can serve http directly using the `--http` option
- uWSGI speaks a native protocol (which nginx also speaks) using `--socket`

It's best to keep the proxy server, especially if you intend to host more than just a single Galaxy server on this system

---
class: middle, top-title, center, largeish
## YAML config

The INI format is limited, structured data formats (e.g. YAML) are more expressive

uWSGI natively supports YAML configs (and INI, and PasteDeploy INI, and XML, and JSON, and ...)

This allows us to put lists and dictionaries directly in to the main config file!

Galaxy configs can now be either INI or YAML

Going forward some new features may only be configurable in YAML

[galaxy.yml.sample](https://github.com/jmchilton/galaxy/blob/78992968ecd0de1f95b99352b53ea2ecd246f954/config/galaxy.yml.sample)

---
class: middle, top-title, largeish

## YAML config

Here's a bit of what's possible:

```yaml
---
uwsgi:
    http: 127.0.0.1:8080

galaxy:
    database_connection: postgresql:///galaxy
    admin_users: >
      nate@bx.psu.edu,
      foo@bar.baz,
      leeroy@jenkins-ci.org
    logging:
        handlers:
            file:
                filename: galaxy.log
        loggers:
            galaxy:
                handlers:
                    - file
```

- YAML line folding is used with `admin_users` to make the value more readable
- `logging` is complex data structure of nested dictionaries and lists

---

## YAML config

*But I don't want to convert my long and tedious `galaxy.ini` to YAML!*

--

Good news! There is a `galaxy.ini`-to-`galaxy.yml` conversion tool:

```sh-session
$ make config-convert-dry-run
$ make config-convert
$ make config-validate
$ make config-lint
```

--

It is now possible to validate and lint the Galaxy config file!

Config names are defined and their values are typed in [config_schema.yml](https://github.com/jmchilton/galaxy/blob/78992968ecd0de1f95b99352b53ea2ecd246f954/lib/galaxy/webapps/galaxy/config_schema.yml)

---
class: middle, top-title

## Configuration Schema

`config_schema.yml` is also the canonical source for config option documentation, from which:
- `galaxy.yml.sample` is generated
- [Galaxy configuration options documentation](https://docs.galaxyproject.org/en/master/admin/config.html) is generated

---
class: middle, top-title

## uWSGI

- Configuration
- **uWSGI wheel**
- Job handler mules

---
class: middle, top-title, largeish

## uWSGI Wheel

- The goal: Provide a no-compilation-required installation method for Galaxy that included uWSGI

--

- Rationale:
  - The Galaxy admin does not need to have a complex build environment on the server
  - We do not need to maintain a list of ever-changing per-distribution build-time dependencies
  - It makes installation fast and failproof (especially for development and CI)
  - We do this for all of Galaxy's other C dependencies

--

- The challenge:
  - Unlike Galaxy's other dependencies, uWSGI is not a Python library
  - uWSGI sits on the front side of Galaxy, rather than the back

---

## uWSGI Wheel

The uWSGI wheel is built differently than when built from source with `pip install uwsgi`

- From source by pip:
  - Built as a single `uwsgi` ELF binary embedding the CPython interpreter
- As a wheel:
  - Built as a Python C extension into an ELF shared library
  - Loaded by a stub `uwsgi` Python script by the CPython interpreter
  - The only way to precompile uWSGI in a way that does not require libpython2.7.so or ship CPython

---
class: middle, top-title, largeish

## uWSGI Wheel

What does this all mean?

Usefulness of the wheel for production servers:
- You aren't redeploying Galaxy repeatedly
- Performing `pip install uwsgi` and waiting 10 seconds is no big deal
- You can also install uWSGI from the system package manager

We have found, reported, and fixed a few bugs with uWSGI-as-shared-library, there are still some outstanding

Installing uWSGI from source via `pip` or the system package manager is still a reasonable approach for production servers

---
class: middle, top-title

## uWSGI

- Configuration
- uWSGI wheel
- **Job handler mules**

---
# A note on processes

uWSGI will run and control a configured number of *anonymous processes* to **serve web requests**

These processes **do not** handle jobs. For this, we start additional processes called *job handlers*

Job handlers are run as [uWSGI Mules](https://uwsgi-docs.readthedocs.io/en/latest/Mules.html)

---
class: largeish

## The Job Handler Problem

In the earliest days of Galaxy, it ran as a single process

As load grew, the largely asynchronous tasks related to handling jobs began to interfere with servicing web requests

--

Our solution was to run multiple Paste, and later, standalone python "webless" Galaxy servers dedicated to job handling

Managing these processes was tedious:
- Add `[server:handlerN]` to `galaxy.ini`
- Add `handlerN` to `job_conf.xml`
- Modify Supervisord config for new handler

Job handlers were assigned at random from the configured options with no regard to their health and notified via the database

--

As production servers shifted to uWSGI for web serving, the configuration became more complex as multiple types of processes needed to be managed

uWSGI presented a unique solution: *Mules*

---
class: middle, top-title

## uWSGI Mules

**Mules** are processes forked from the uWSGI master after the application has been loaded

Mules can continue to run the same code or can load and run arbitrary code

Mules can receive messages from the web proceses

Mules can be pooled in to *Farms*, and messages can be sent to the farm to be handled by any mule in that farm

This design made mules ideally suited to perform as Galaxy job handlers

---
class: middle, top-title

## Mule Advantages

- Mules are spawned by the uWSGI *master process* and therefore are killed with it
- Due to farm messaging, mules can self-assign jobs

Thus, with mules **there is no multiprocess management complexity** and **no job conf changes are required**

In addition, **nonresponsive mules cannot be assigned jobs**

---
class: middle, top-title

## Job Handler Mule Configuration

Adding job handler mules is performed by simply instructing uWSGI to start them in `galaxy.yml`:

```yaml
uwsgi:
    mule: lib/galaxy/main.py
    mule: lib/galaxy/main.py
    farm: job-handlers:1,2
```

That's it! This Galaxy instance will now start and use two job handler mules.

---
class: middle, top-title, largeish

## How Mules Work

1. Master process loads Galaxy
2. Master `fork()`s web worker processes
3. Master `fork()`s job handler mules
4. Mules reload Galaxy as job handlers
5. The first mule to fully initialize grabs a lock on the message queue
6. All additional mules wait on the lock
7. A web worker receives a request for a new job
8. The web worker creates a `job` record in the database but leaves the `handler` field `null`
9. The web worker uses uWSGI's `farm_msg()` function to notify the `job-handlers` farm that a new job is ready to run
10. The mule with the lock receives the message, assigns itself, and gives up the lock
11. Another mule acquires the lock and waits for the next message
12. Job handling continues as normal

---
class: middle, top-title, center

## Mule Messaging Implementation

Mule messaging is simply a socket open between the processes - it is up to you to implement the protocol

For this, we developed a generic JSON-based message format and routing system so that any Galaxy component can utilize mules

This leaves many possibilities for offloading work across Galaxy

---
class: middle, top-title, center

## Job Configuration

All of the previous handler-mapping functionality is still supported with mules

Tools can be statically or dynamically mapped to specific mules
