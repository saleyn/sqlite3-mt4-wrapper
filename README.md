sqlite3-mt4 - An sqlite3 binding for MT4,5
==========================================

This is a non-backward compatible fork with the original author's
[repository](https://github.com/Shmuma/sqlite3-mt4-wrapper).
It is written in C++ and contains many enhancements and bug fixes
and is maintained separately from the origin.  If you are happy
with the original C implementation, you should use the other
project, but if you need an object-oriented library that is
stable and tested, use this one.

## Howto

1. Download [latest release](https://github.com/saleyn/sqlite3-mt4/releases)
2. Follow instructions in the release to install files in the ``MQL`` sandbox directory,
   ``<TERMINAL_DATA_PATH>/MQL{4,5}`` (depending on use of MT4 or MT5 terminal).
MT4's:
```
MQL4/Include/sqlite.mqh
MQL4/Libraries/MQT/mqt-sqlite3.x86.dll
```
and MT5's:
```
MQL5/Include/sqlite.mqh
MQL5/Libraries/MQT/mqt-sqlite3.x64.dll
```

3. In your EA/Indicator/Script, add the following include file:

```cpp
#include <sqlite.mqh>
```
4. Here is a "real-life" example of reading trade records in MT4 from a DB file:

```cpp
void Test(string path_to_dbfile) {
  SQLite db;
  if (!db.OpenReadOnly(path_to_dbfile)) {
    Alert(StringFormat("Cannot open database %s: %s", path_to_dbfile, db.ErrMsg()));
    return;
  }

  string sql = StringFormat(
                  "SELECT "
                  "  ticket,ts_open,lots,px_open,sl,tp,ts_close,px_close,"
                  "  magic,fees,swap,profit,comment,type "
                  "FROM trade "
                  "WHERE symbol like '%s%%' and type < 2 "
                  "ORDER BY ts_open",
                  _Symbol
               );
  
  SQLiteQuery* query = db.Prepare(sql);
  if (!query) {
    Alert(StringFormat("Invalid query: %s", db.ErrMsg()));
    return;
  }

  int pip_points = PipPoints(_Symbol);
  int count      = 0;
  int tzoff      = (int)(TimeCurrent()-TimeGMT());

  for(; query.Next() == SQLITE_ROW; ++count) {
    int      ticket   = query.GetInt(0);
    datetime ts_open  = query.GetDatetime(1)+tzoff;
    double   lots     = query.GetDouble(2);
    double   px_open  = query.GetDouble(3);
    double   sl       = query.GetDouble(4);
    double   tp       = query.GetDouble(5);
    datetime ts_close = query.GetDatetime(6)+tzoff;
    double   px_close = query.GetDouble(7);
    double   fees     = query.GetDouble(9);
    double   swap     = query.GetDouble(10);
    double   profit   = query.GetDouble(11);
    string   comment  = query.GetString(12);
    int      type     = query.GetInt(13);

    double   tot_prof = fees+swap+profit;
    double   pips     = MathAbs(px_close-px_open)/_Point/pip_points;

    string   side     = type < 1 ? "buy" : "sell";
    PrintFormat("#%d %s %.2f at %.5f close at %.5f (%.1fp) %s $%.2f%s"),
                ticket, side, lots,
                px_open, px_close, pips,
                tot_prof >= 0 ? "profit" : "loss", tot_prof,
                StringLen(comment) ? "" : ", cmt="+comment);
  }

  delete query;

  int res = db.ExecDDL(StringFormat(
                "INSERT INTO balance (date,balance,equity,free_margin)"
                "VALUES (%d,%.2f,%.2f,%.2f)",
                TimeLocal(), AccountBalance(), AccountEquity(), AccountFreeMargin()));
  if (res != SQLITE_OK)
    PrintFormat("Couldn't write balance record to database: (%d) %s", db.ErrCode(), db.ErrMsg());

  query = db.Prepare("INSERT INTO balance (date,balance,equity,free_margin)"
                     "VALUES (?,?,?,?)");
  if (!query)
    PrintFormat("Error parsing query: (%d) %s", db.ErrCode(), db.ErrMsg()
  int col=0;
  query.Reset();
  query.Bind(++col, TimeLocal());
  query.Bind(++col, AccountBalance());
  query.Bind(++col, AccountEquity());
  query.Bind(++col, AccountFreeMargin());
  if (!upd_query.Exec())
    PrintFormat("Failed to insert update record: (%d) %s", db.ErrCode(), db.ErrMsg());

  delete query;
}
```

## Database file

Database file is by default stored to ``<TERMINAL_DATA_PATH>\MQL4\Files\SQLite``.

If you specify a full path as database filename, it's used.

## Terminal data path

TERMINAL_DATA_PATH is the location of MetaTrader's sandbox directory that can be
obtained by the following steps:

1. Open MT4
2. Open [File] menu
3. Click "Open Data Folder"

## Sample

