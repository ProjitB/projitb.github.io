# Dynamically Importing code in Python

When we write python code, we often split it up into multiple files in the name of "modularity" and "clean code". But this comes with this clearly **unrealistic** expectation of having all your files on the pythonpath. Ridiculous isn't it?

Let's write a custom importer to handle finding our source files from arbitrary (but I guess predefined?) locations. We'll also then extend our system to import (and use) new code at runtime.

Contents:
- [Tools](#tools)
- [Custom Importer](#custom-importer)
- [Import at Runtime](#import-at-runtime)
- [Conclusion](#conclusion)


## Tools
Quickstart on vagrant setup for this:
``` bash
> vagrant init ubuntu/bionic64
> vagrant up
> vagrant ssh
```

Though it should come in-built, we do need python for this

## Custom Importer

Let's put up some rules. For the modules we want to import using our custom logic, let's say that they are at path: custom
```python
# file: main.py
import sys
from importer import FindImports
sys.meta_path.append(FindImports)
from custom.somefile import external_func

def main():
    external_func(10)

if __name__ == '__main__':
    main()

```

Let's also define this somefile, but we're not going to put it in the path custom. It'll be somewhere, let's say on a github gist.
```python
# file: somefile.py   located: https://gist.githubusercontent.com/ProjitB/170993eb36fa7f23152c745a36e63cfc/raw/d5577826bde341ab763688c3d7ab7a5d7848fb0b/somefile.py
def external_func(arg):
  print("the arg you passed is {}".format(str(arg)))

```

```python
# file: importer.py
import sys
import types
import logging
import requests
logging.basicConfig(level=logging.DEBUG)

class FindImports(object):
    @staticmethod
    def find_module(name, path=None):
        logging.debug("Custom Importer Invoked for {}".format(name))
        try:
            paths = name.split('.')
            if paths[0] != 'custom':
                return
            if len(paths) == 1:
                is_pkg = True
                code = ''
            else:
                is_pkg = False
                fname = paths[-1]
                # Implement our search logic for this file now
                # Using extremely naiive one
                code = get_code(fname)
            return ImportLoader(name, path, code, is_pkg)
        except Exception as e:
            logger.debug("Had an exception. Probably couldn't find the module. {}".format(e))

class ImportLoader(object):
    def __init__(self, name, path, code, is_pkg):
        self.name = name
        self.path = path
        self.code = code
        self.is_pkg = is_pkg

    def load_module(self, name):
        mod = sys.modules.setdefault(name, types.ModuleType(name))
        mod.__file__ = "<{}>".format(self.__class__.__name__)
        mod.__loader__ = self
        if self.is_pkg:
            mod.__path__ = []
            mod.__package__ = name
        else:
            mod.__package__ = name.rpartition('.')[0]
            exec(self.code, mod.__dict__)
        return mod
```

The find\_code function can essentially search anywhere in the world for the code. It's responsibility is to find the file and return the source code.
```python
# file: importer.py
def find_code(filename):
    if filename == 'somefile':
        url = 'https://gist.githubusercontent.com/ProjitB/170993eb36fa7f23152c745a36e63cfc/raw/d5577826bde341ab763688c3d7ab7a5d7848fb0b/somefile.py'
        response = requests.get(url, verify=False)
        return response.text
    else:
        raise Exception("File can't be searched")
```


Now if we run main.py we see:
``` bash
> python main.py
DEBUG:root:Custom Importer Invoked for custom
DEBUG:root:Custom Importer Invoked for custom.somefile
DEBUG:urllib3.connectionpool:Starting new HTTPS connection (1): gist.githubusercontent.com
DEBUG:urllib3.connectionpool:https://gist.githubusercontent.com:443 "GET /ProjitB/170993eb36fa7f23152c745a36e63cfc/raw/d5577826bde341ab763688c3d7ab7a5d7848fb0b/somefile.py HTTP/1.1" 200 76
the arg you passed is 10
```

Unfortunately the gist urls do update with each edit. However if you were to use a repository instead, file locations would be constant. You could update your remote code, and your local implementation will always go and fetch the latest.
For all intents and purposes, this importing structure is indistinguishable to all other functions within python. Try it out!

## Import at Runtime
Given the previous part, this is actually quite trivial.
An example?
```python
# Alternate main.py
import sys
from importer import FindImports
sys.meta_path.append(FindImports)

def main():
    inp = input("Enter the file you want to import: ")
    if inp == 'somefile':
        mod = __import__("custom.{}".format(inp), fromlist=['object'])
        func = getattr(mod, 'external_func')
        func(3)
    else:
        print("Can't import that file, sorry")

if __name__ == '__main__':
    main()
```
Running it:
```bash
> python3 main.py
Enter the file you want to import: abc
Can't import that file, sorry
> python3 main.py
Enter the file you want to import: somefile
DEBUG:root:Custom Importer Invoked for custom
DEBUG:root:Custom Importer Invoked for custom.somefile
DEBUG:urllib3.connectionpool:Starting new HTTPS connection (1): gist.githubusercontent.com
DEBUG:urllib3.connectionpool:https://gist.githubusercontent.com:443 "GET /ProjitB/170993eb36fa7f23152c745a36e63cfc/raw/d5577826bde341ab763688c3d7ab7a5d7848fb0b/somefile.py HTTP/1.1" 200 76
the arg you passed is 3
```

Run this, and you'll see that at runtime, the file gets imported. The code extracts a known attribute from this, which is the function, and then runs this.


## Conclusion
 Our toy example imports 'somefile'. Instead of hardcoding the import to that particular url, we could've written a small searching function, to actually go through avaliables public gists on my account, and decide which one is correct (wouldn't be hard, I'm just a bit lazy :) ). Our main function importing code based on some input is essentially dynamically imported code.

 On the whole, you see that if you have well defined function names / classes in these dynamically imported files, you can actually created a pretty scalable structure.


## References:
- [1] [https://www.python.org/dev/peps/pep-0302/](https://www.python.org/dev/peps/pep-0302/)
