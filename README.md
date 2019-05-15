# transfer-via-ssh-tunnel
transfer mongoDB data via SSH tunnel
```python
from sshtunnel import SSHTunnelForwarder
import pymongo
import gridfs
import pymysql
from bson.objectid import ObjectId

if __name__ == '__main__':
    local_mysql_db = pymysql.connect(
        "192.168.1.127", 
        "username", 
        "password", 
        "table_name", 
        charset='utf8', 
        cursorclass=pymysql.cursors.SSDictCursor)
    local_mysql_cursor = local_mysql_db.cursor()
    local_mongo_conn = pymongo.MongoClient('192.168.1.127', 27017) # local mongodb address
    local_mongo_db = local_mongo_conn["mongo_database_name"]
    local_gfs = gridfs.GridFS(local_mongo_db, collection="fs")
    local_mysql_cursor.execute("SELECT DISTINCT(logo_url) FROM organization WHERE logo_url IS NOT NULL") # query all record which needs transfer
    
    # define ssh tunnel
    server = SSHTunnelForwarder(
        ('45.77.xxx.xxx', 22), # remote ssh server address
        ssh_username="nobrowning", # ssh username
        ssh_password="123456", # ssh password
        remote_bind_address=('192.168.0.84', 27017)# remote mongodb address
    )
    # start ssh tunnel
    server.start()
    remote_mongo_conn = pymongo.MongoClient('127.0.0.1', server.local_bind_port) # must be 127.0.0.1
    remote_mongo_db = remote_mongo_conn["mongo_database_name"]
    remote_gfs = gridfs.GridFS(remote_mongo_db, collection="fs")

    # transfer
    for file_name in local_mysql_cursor.fetchall():
        name = str(file_name['logo_url'])
        name = name[0: name.find('.')]
        file = local_gfs.find_one({"_id": ObjectId(name)})
        if not remote_gfs.exists(document_or_id=ObjectId(name)):
            remote_gfs.put(file, _id=ObjectId(name), filename=file.filename, contentType=file.content_type, metadata=file.metadata)

    # close ssh tunnel
    remote_mongo_conn.close()
    local_mongo_conn.close()
    local_mysql_db.close()
    server.stop()
```
