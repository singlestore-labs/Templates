# <img src="https://github.com/singlestore-labs/singlestoredb-python/blob/main/resources/singlestore-logo.png" height="60" valign="middle"/> A serverless architecture for creating embeddings with SingleStoreDB

This project contains a [DB-API 2.0](https://www.python.org/dev/peps/pep-0249/)
compatible Python interface to the SingleStore database and workspace management API.

> **Warning**
> As of version v0.5.0, the parameter substitution syntax has changed from `:1`, `:2`, etc.
> for list parameters and `:foo`, `:bar`, etc. for dictionary parameters to `%s` and `%(foo)s`,
> `%(bar)s` etc., respectively.

## Install

This package can be install from PyPI using `pip`:
```
pip install singlestoredb
```

