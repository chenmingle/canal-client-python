# canal-client-python
* 环境要求
	* python >= 3

* 构建canal python客户端

	```
	pip install canal-python
	```

* 建立与Canal的连接


	```
	#!/usr/bin/env python
	# -*- coding: utf-8 -*-
	__author__ = 'chenmingle'
	
	import time
	import sys
	try:
	    from canal.client import Client
	    from canal.protocol import EntryProtocol_pb2
	    from canal.protocol import CanalProtocol_pb2
	except Exception as e:
	    print('pip install canal-python')
	    sys.exit(1)
	
	try:
	    import MySQLdb
	except Exception as e:
	    print('pip install MySQLdb')
	    sys.exit(1)
	    
	
	
	# 打开数据库连接
	db = MySQLdb.connect("127.0.0.1", "root", "test123", "test2db", charset='utf8' )
	
	# 使用cursor()方法获取操作游标
	cursor = db.cursor()
	localtime = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
	
	client = Client()
	client.connect(host='192.168.1.1', port=11111)
	client.check_valid(username=b'', password=b'')
	client.subscribe(client_id=b'106', destination=b'example', filter=b'testdb\\.test,testdb\\.test2')
	
	def deleteRow(table, id):
	    sql = "DELETE FROM %s WHERE id=%s" % (table, id)
	    print(localtime)
	    print(sql)
	    try:
	        # 执行SQL语句
	        cursor.execute(sql)
	        # 提交修改
	        db.commit()
	    except:
	        # 发生错误时回滚
	        db.rollback()
	
	def insertRow(table, keys, values):
	    sql = "INSERT ignore INTO %s (%s) VALUES (%s)" % (table, keys, values)
	    print(localtime)
	    print(sql)
	    try:
	        # 执行SQL语句
	        cursor.execute(sql)
	        # 提交修改
	        db.commit()
	    except:
	        # 发生错误时回滚
	        db.rollback()
	
	def updateRow(table, values, id):
	    # print(values)
	    # print(id)
	    sql = "update %s set %s where id=%s" % (table, values, id)
	    print(localtime)
	    print(sql)
	    try:
	        # 执行SQL语句
	        cursor.execute(sql)
	        # 提交修改
	        db.commit()
	    except:
	        # 发生错误时回滚
	        db.rollback()
	
	while True:
	    message = client.get(100)
	    entries = message['entries']
	    for entry in entries:
	        entry_type = entry.entryType
	        if entry_type in [EntryProtocol_pb2.EntryType.TRANSACTIONBEGIN, EntryProtocol_pb2.EntryType.TRANSACTIONEND]:
	            continue
	        row_change = EntryProtocol_pb2.RowChange()
	        row_change.MergeFromString(entry.storeValue)
	        event_type = row_change.eventType
	        header = entry.header
	        database = header.schemaName
	        table = header.tableName
	        event_type = header.eventType
	        for row in row_change.rowDatas:
	            format_data = dict()
	            if event_type == EntryProtocol_pb2.EventType.DELETE:
	                # print(row.beforeColumns)
	                for column in row.beforeColumns:
	                    if column.name == "id":
	                        format_data = {
	                            column.name: column.value
	                        }
	                id = format_data["id"]
	                deleteRow(table,id)
	            elif event_type == EntryProtocol_pb2.EventType.INSERT:
	                # print(row.afterColumns)
	                keys_d=[]
	                values_d=[]
	                for column in row.afterColumns:
	                    format_data = {
	                        column.name: column.value
	                    }
	                    keys_d.append(column.name)
	                    values_d.append(column.value)
	                keys = ','.join(str(d) for d in keys_d)
	                values = ','.join(str(v) for v in values_d)
	                insertRow(table, keys, values)
	            else:
	                values_d = []
	                format_data['before'] = format_data['after'] = dict()
	                for column in row.beforeColumns:
	                    format_data['before'][column.name] = column.value
	                for column in row.afterColumns:
	                    if column.name == "id":
	                        id = column.value
	                    format_data['after'][column.name] = column.value
	                    v_data = column.name+'='+column.value
	                    values_d.append(v_data)
	                values = ','.join(str(d) for d in values_d)
	                updateRow(table, values, id)
	            data = dict(
	                db=database,
	                table=table,
	                event_type=event_type,
	                data=format_data,
	            )
	            # print("操作表：%s" % (data['table']))


	
	time.sleep(1)
	
	client.disconnect()
	
	
	```

* 执行脚本实时同步增删改查

```
python canal-python/canal_client_python.py
connected to 192.168.1.1:11111
Auth succed
Subscribe succed
2020-03-17 16:24:17
update test set id=1,name=test1,age=11 where id=1
2020-03-17 16:24:18
update test2 set id=2,name=test2,age=22 where id=2
```
