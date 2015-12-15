Use NDBC API to access ORACLE
-----------------------------

through PL/SQL servlet can print arbitrary content to feed NDBC call,
the most NDBC use is for fetch SQL 
[result sets](https://github.com/kaven276/noradle/wiki/SQL-resultsets), 
so the example is for that.

### The node javascript client who call in oracle plsql servlet
```javascript
var DBDriver = require('noradle-nodejs-client')
  , NDBC = require('noradle-ndbc')
  , dbPool = DBDriver.connect([9009], {cid : 'test', passwd : 'test'})
  , dbc = new NDBC.DBCall(dbPool, {__parse : true, 'x$dbu': 'demo'})
  ;
dbc.call('db_src_b.example', {limit : 3}, function(status, headers, page){
  if(status!==200) {
    ...
    return;
  }
  console.log(status);
  console.log(headers);
  console.log(page);
});
```
Note:
* NDBC instance have dbPool as oracle access path supporting object as `(dbPool, {...`
* NDBC have default parameter in second parameter as `{__parse : true, 'x$dbu': 'demo'}`
* ndbc.call(stored_procedure_name, parameter, cb) will combine default parameter to make a PL/SQL servlet request
* callback(status, headers, page) is just like http response
* headers is name-value(s) pairs
* page is Buffer, String or Object type as headers['content-type'] indicate
* `__parse : true` will let noradle automatically convert certain response type to parsed javascript objects

for how raw SQL result is parsed, see [noradle-resultsets][]

  [noradle-resultsets]: https://github.com/noradle/noradle-resultsets

### The oracle plsql sevlet code who generate SQL result sets
```plsql
create or replace package body db_src_b is

    procedure example is
        cur sys_refcursor;
        v1  varchar2(50) := 'psp.web';
        v2  number := 123456;
        v3  date := date '1976-10-26';
    begin
        h.content_type('text/resultsets');

        open cur for
            select a.object_name, a.subobject_name, a.object_type, a.created
                from user_objects a
             where rownum <= r.getn('limit', 8);
        rs.print('test', cur);

        open cur for
            select v1 as name, v2 as val, v3 as ctime from dual;
        rs.print('namevals', cur);
    end;

end db_src_b;
```
Note:
* `r.getn('limit', 8)` will get number type parameter `limit`, as js code, it's 3, and default to `8` if no this input
* `h.content_type('text/resultsets');` specify the response content-type is result sets 
 that can be converted to javascript object or JSON text
* `rs.print(name, sys_refcursor)` will print SQL result and its meta data to condensed table format
* `v1,v2,v3` is varchar2/number/date types, all scalar data can be printed out with `from dual` SQL

### the data in transfer
```
[objects]
OBJECT_NAME:1,SUBOBJECT_NAME:1,OBJECT_TYPE:1,CREATED:12
TOOL,,TYPE,2015-04-20 16:38:39
TOOL2,,TYPE,2015-04-20 16:38:39
TERM_T,,TABLE,2015-04-20 16:38:45

[namevals]
NAME:1,VAL:2,CTIME:12,P1:1,P2:1,PNULL:1
psp.web,123456,1976-10-26 00:00:00,value1,value2,
```
Note:
* SQL resultsets is printed section by section
* one resultset have one meta line and zero, one or more data lines each line for one record
* columns/lines are separate not only with comma and linefeed, but with a hidden ACSII char together with separator
 so column content can safely have comma and linefeed.

### the output
```
200
{ Date: 'Fri, 24 Jul 2015 00:55:46 GMT',
  'Content-Encoding': '?',
  'Content-Length': '649',
  'Content-Type': 'text/resultsets; charset=UTF-8',
  'x-pw-timespan': '40 / 40 ms' }
{ objects: 
   { name: 'objects',
     attrs: 
      [ { name: 'object_name', dataType: 1 },
        { name: 'subobject_name', dataType: 1 },
        { name: 'object_type', dataType: 1 },
        { name: 'created', dataType: 12 } ],
     rows: 
      [ { object_name: 'TOOL',
          subobject_name: '',
          object_type: 'TYPE',
          created: '2015-04-20 16:38:39' },
        { object_name: 'TOOL2',
          subobject_name: '',
          object_type: 'TYPE',
          created: '2015-04-20 16:38:39' },
        { object_name: 'TERM_T',
          subobject_name: '',
          object_type: 'TABLE',
          created: '2015-04-20 16:38:45' } ] },
  namevals: 
   { name: 'namevals',
     attrs: 
      [ { name: 'name', dataType: 1 },
        { name: 'val', dataType: 2 },
        { name: 'ctime', dataType: 12 },
        { name: 'p1', dataType: 1 },
        { name: 'p2', dataType: 1 },
        { name: 'pnull', dataType: 1 } ],
     rows: 
      [ { name: 'psp.web',
          val: 123456,
          ctime: '1976-10-26 00:00:00',
          p1: 'value1',
          p2: 'value2',
          pnull: '' } ] } }
```
Note:
* the parsed final result is javascript object, each key stand for one result set
* each result set have name, attrs(column meta data) array and rows array