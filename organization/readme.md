# UniKey

This repository contains a reveal.js presentation for how we might change how we organize teams at UniKey.

---

### Running the site

Reveal.js requires the presentation to be hosted on an HTTP server to work properly.  You can run any http server from [this list](https://gist.github.com/willurd/5720255) in this directory and then visit http://localhost:8000 to see the presentation.

Some specific examples would be with Ruby (which ships on all Macs):

```
ruby -run -ehttpd . -p8000
```

Or in Python 2 (ships on all Linux distros)

```
python -m SimpleHTTPServer 8000
```

