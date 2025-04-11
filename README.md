 ==================================== ERRORS ====================================
01:59:54   ____________ ERROR collecting test/Services/test_fileoperations.py _____________
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:2336: in _wrap_pool_connect
01:59:54       return fn()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:304: in unique_connection
01:59:54       return _ConnectionFairy._checkout(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:778: in _checkout
01:59:54       fairy = _ConnectionRecord.checkout(pool)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:495: in checkout
01:59:54       rec = pool._do_get()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/impl.py:140: in _do_get
01:59:54       self._dec_overflow()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/langhelpers.py:68: in __exit__
01:59:54       compat.raise_(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/compat.py:182: in raise_
01:59:54       raise exception
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/impl.py:137: in _do_get
01:59:54       return self._create_connection()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:309: in _create_connection
01:59:54       return _ConnectionRecord(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:440: in __init__
01:59:54       self.__connect(first_connect_check=True)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:661: in __connect
01:59:54       pool.logger.debug("Error on connect(): %s", e)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/langhelpers.py:68: in __exit__
01:59:54       compat.raise_(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/compat.py:182: in raise_
01:59:54       raise exception
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:656: in __connect
01:59:54       connection = pool._invoke_creator(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/strategies.py:114: in connect
01:59:54       return dialect.connect(*cargs, **cparams)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/default.py:508: in connect
01:59:54       return self.dbapi.connect(*cargs, **cparams)
01:59:54   E   pyodbc.OperationalError: ('HYT00', '[HYT00] [Microsoft][ODBC Driver 17 for SQL Server]Login timeout expired (0) (SQLDriverConnect)')
01:59:54   
01:59:54   The above exception was the direct cause of the following exception:
01:59:54   test/Services/test_fileoperations.py:7: in <module>
01:59:54       import Services.fileoperations as fo
01:59:54   Services/fileoperations.py:11: in <module>
01:59:54       dbops_obj = dbops.dboperations()
01:59:54   Services/dboperations.py:28: in __init__
01:59:54       self.connection = self.engine.connect()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:2263: in connect
01:59:54       return self._connection_cls(self, **kwargs)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:104: in __init__
01:59:54       else engine.raw_connection()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:2369: in raw_connection
01:59:54       return self._wrap_pool_connect(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:2339: in _wrap_pool_connect
01:59:54       Connection._handle_dbapi_exception_noconnection(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:1583: in _handle_dbapi_exception_noconnection
01:59:54       util.raise_(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/compat.py:182: in raise_
01:59:54       raise exception
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/base.py:2336: in _wrap_pool_connect
01:59:54       return fn()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:304: in unique_connection
01:59:54       return _ConnectionFairy._checkout(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:778: in _checkout
01:59:54       fairy = _ConnectionRecord.checkout(pool)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:495: in checkout
01:59:54       rec = pool._do_get()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/impl.py:140: in _do_get
01:59:54       self._dec_overflow()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/langhelpers.py:68: in __exit__
01:59:54       compat.raise_(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/compat.py:182: in raise_
01:59:54       raise exception
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/impl.py:137: in _do_get
01:59:54       return self._create_connection()
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:309: in _create_connection
01:59:54       return _ConnectionRecord(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:440: in __init__
01:59:54       self.__connect(first_connect_check=True)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:661: in __connect
01:59:54       pool.logger.debug("Error on connect(): %s", e)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/langhelpers.py:68: in __exit__
01:59:54       compat.raise_(
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/util/compat.py:182: in raise_
01:59:54       raise exception
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/pool/base.py:656: in __connect
01:59:54       connection = pool._invoke_creator(self)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/strategies.py:114: in connect
01:59:54       return dialect.connect(*cargs, **cparams)
01:59:54   /usr/local/lib/python3.9/dist-packages/sqlalchemy/engine/default.py:508: in connect
01:59:54       return self.dbapi.connect(*cargs, **cparams)
01:59:54   E   sqlalchemy.exc.OperationalError: (pyodbc.OperationalError) ('HYT00', '[HYT00] [Microsoft]
