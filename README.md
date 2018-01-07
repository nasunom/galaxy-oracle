# Galaxy の Oracle Database 対応メモ

## Oracle Instant Client インストール

```
$ tail ~galaxy/.bash_profile 

export ORACLE_HOME=/usr/lib/oracle/12.2/client64
export LD_LIBRARY_PATH=/usr/lib/oracle/12.2/client64/lib:$LD_LIBRARY_PATH
```

## galaxy.ini の DB 接続パラメータ

```
$ diff config/galaxy.ini.sample config/galaxy.ini
> database_connection = oracle://galaxy:galaxy@127.0.0.1:1521/galaxy
```

## cx_Oracle インストール設定

Galaxy 起動時に pip install されるように設定する。

```
$ tail galaxy/requirements.txt 
# cx_Oracle
#cx_oracle==5.3
cx_oracle==6.1
```

## 起動時エラー対応メモ

### Unicode 関連エラー

ひとまず、SQLAlchemy のソースを修正してエラーを回避してみる。

```
$ diff -u .venv/lib/python2.7/site-packages/sqlalchemy/dialects/oracle/cx_oracle.py.orig .venv/lib/python2.7/site-packages/sqlalchemy/dialects/oracle/cx_oracle.py
--- .venv/lib/python2.7/site-packages/sqlalchemy/dialects/oracle/cx_oracle.py.orig	2018-01-07 11:38:48.943611442 +0900
+++ .venv/lib/python2.7/site-packages/sqlalchemy/dialects/oracle/cx_oracle.py	2018-01-07 11:41:19.479908045 +0900
@@ -716,38 +716,8 @@
 
         self._cx_oracle_native_nvarchar = self.cx_oracle_ver >= (5, 0)
 
-        if self.cx_oracle_ver is None:
-            # this occurs in tests with mock DBAPIs
-            self._cx_oracle_string_types = set()
-            self._cx_oracle_with_unicode = False
-        elif util.py3k or (
-                self.cx_oracle_ver >= (5,) and not \
-                hasattr(self.dbapi, 'UNICODE')
-        ):
-            # cx_Oracle WITH_UNICODE mode.  *only* python
-            # unicode objects accepted for anything
-            self.supports_unicode_statements = True
-            self.supports_unicode_binds = True
-            self._cx_oracle_with_unicode = True
-
-            if util.py2k:
-                # There's really no reason to run with WITH_UNICODE under
-                # Python 2.x.  Give the user a hint.
-                util.warn(
-                    "cx_Oracle is compiled under Python 2.xx using the "
-                    "WITH_UNICODE flag.  Consider recompiling cx_Oracle "
-                    "without this flag, which is in no way necessary for "
-                    "full support of Unicode. Otherwise, all string-holding "
-                    "bind parameters must be explicitly typed using "
-                    "SQLAlchemy's String type or one of its subtypes,"
-                    "or otherwise be passed as Python unicode.  "
-                    "Plain Python strings passed as bind parameters will be "
-                    "silently corrupted by cx_Oracle."
-                )
-                self.execution_ctx_cls = \
-                    OracleExecutionContext_cx_oracle_with_unicode
-        else:
-            self._cx_oracle_with_unicode = False
+        self._cx_oracle_string_types = set()
+        self._cx_oracle_with_unicode = False
 
         if self.cx_oracle_ver is None or \
                 not self.auto_convert_lobs or \
```

### cx_Oracle と SQLAlchemy のバージョン競合

cx_Oracle v6.0 で `twophase` キーワードが廃止されたことによるエラー。

```
$ sh run.sh
   :

Traceback (most recent call last):
  File "/home/galaxy/galaxy/lib/galaxy/webapps/galaxy/buildapp.py", line 58, in paste_app_factory
    app = galaxy.app.UniverseApplication(global_conf=global_conf, **kwargs)
  File "/home/galaxy/galaxy/lib/galaxy/app.py", line 76, in __init__
    self._configure_models(check_migrate_databases=True, check_migrate_tools=check_migrate_tools, config_file=config_file)
  File "/home/galaxy/galaxy/lib/galaxy/config.py", line 1045, in _configure_models
    create_or_verify_database(db_url, config_file, self.config.database_engine_options, app=self)
  File "/home/galaxy/galaxy/lib/galaxy/model/migrate/check.py", line 52, in create_or_verify_database
    Table("dataset", meta, autoload=True)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/sql/schema.py", line 440, in __new__
    metadata._remove_table(name, schema)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/util/langhelpers.py", line 60, in __exit__
    compat.reraise(exc_type, exc_value, exc_tb)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/sql/schema.py", line 435, in __new__
    table._init(name, metadata, *args, **kw)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/sql/schema.py", line 510, in _init
    self._autoload(metadata, autoload_with, include_columns)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/sql/schema.py", line 534, in _autoload
    self, include_columns, exclude_columns
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/engine/base.py", line 1971, in run_callable
    with self.contextual_connect() as conn:
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/engine/base.py", line 2039, in contextual_connect
    self._wrap_pool_connect(self.pool.connect, None),
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/engine/base.py", line 2074, in _wrap_pool_connect
    return fn()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 376, in connect
    return _ConnectionFairy._checkout(self)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 713, in _checkout
    fairy = _ConnectionRecord.checkout(pool)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 480, in checkout
    rec = pool._do_get()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 1060, in _do_get
    self._dec_overflow()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/util/langhelpers.py", line 60, in __exit__
    compat.reraise(exc_type, exc_value, exc_tb)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 1057, in _do_get
    return self._create_connection()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 323, in _create_connection
    return _ConnectionRecord(self)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 449, in __init__
    self.connection = self.__connect()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/pool.py", line 607, in __connect
    connection = self.__pool._invoke_creator(self)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/engine/strategies.py", line 97, in connect
    return dialect.connect(*cargs, **cparams)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/sqlalchemy/engine/default.py", line 385, in connect
    return self.dbapi.connect(*cargs, **cparams)
TypeError: 'twophase' is an invalid keyword argument for this function
```

⇒ v5.3 の cx_Oracle をソースからビルドして
`.venv/lib/python2.7/site-packages/cx_Oracle.so` を置き換え。

なお、v5.3 のビルドには Oracle Instance Client SDK が必要。

```
$ rpm -qa|grep oracle
oracle-instantclient12.2-basic-12.2.0.1.0-1.x86_64
oracle-instantclient12.2-devel-12.2.0.1.0-1.x86_64
```

### Migration スクリプト

```
$ sh run.sh
   :
migrate.versioning.repository DEBUG 2018-01-07 12:10:43,680 Config: OrderedDict([('db_settings', OrderedDict([('__name__', 'db_settings'), ('repository_id', 'Galaxy'), ('version_table', 'migrate_version'), ('required_dbs', "['oracle']")]))])
galaxy.model.migrate.check INFO 2018-01-07 12:10:43,799 No database, initializing
Traceback (most recent call last):
  File "/home/galaxy/galaxy/lib/galaxy/webapps/galaxy/buildapp.py", line 58, in paste_app_factory
    app = galaxy.app.UniverseApplication(global_conf=global_conf, **kwargs)
  File "/home/galaxy/galaxy/lib/galaxy/app.py", line 76, in __init__
    self._configure_models(check_migrate_databases=True, check_migrate_tools=check_migrate_tools, config_file=config_file)
  File "/home/galaxy/galaxy/lib/galaxy/config.py", line 1045, in _configure_models
    create_or_verify_database(db_url, config_file, self.config.database_engine_options, app=self)
  File "/home/galaxy/galaxy/lib/galaxy/model/migrate/check.py", line 59, in create_or_verify_database
    migrate()
  File "/home/galaxy/galaxy/lib/galaxy/model/migrate/check.py", line 41, in migrate
    db_schema = schema.ControlledSchema(engine, migrate_repository)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/migrate/versioning/schema.py", line 33, in __init__
    self.load()
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/migrate/versioning/schema.py", line 54, in load
    exceptions.DatabaseNotControlledError(str(exc)), tb)
  File "/home/galaxy/galaxy/.venv/lib/python2.7/site-packages/migrate/versioning/schema.py", line 50, in load
    data = list(result)[0]
DatabaseNotControlledError: list index out of range
```

⇒ おそらく `lib/galaxy/model/migrate/` 配下の対応が必要。

```
$ git diff lib/galaxy/model/migrate/migrate.cfg 
diff --git a/lib/galaxy/model/migrate/migrate.cfg b/lib/galaxy/model/migrate/migrate.cfg
index 3fd7400..031e424 100644
--- a/lib/galaxy/model/migrate/migrate.cfg
+++ b/lib/galaxy/model/migrate/migrate.cfg
@@ -17,4 +17,4 @@ version_table=migrate_version
 # entire commit will fail. List the databases your application will actually 
 # be using to ensure your updates to that database work properly.
 # This must be a list; example: ['postgres','sqlite']
-required_dbs=[]
+required_dbs=['oracle']

$ cd lib/galaxy/model/migrate/versions && sed -i 's/postgresql/oracle/g' 0003_security_and_libraries.py
```
⇒ これだけでは効果無し。

### その他、既知の問題

Oracle のテーブル名は「30 バイト以下」の制限があるため、Galaxyで使用するテーブル名をこれに合わせる必要がある。
`lib/galaxy/model/migrate/versions/????_*_tables.py` を機械的に修正することで対応できるかも？
