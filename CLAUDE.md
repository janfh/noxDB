# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is noxDB

noxDB is an IBM i (AS/400) ILE C framework enabling RPG developers to work with XML, JSON, and SQL via a unified in-memory object graph. All data—parsed JSON, XML, and SQL result sets—is represented as a node tree navigable using XPath-like expressions.

**Important branch note**: The `master` branch stores the core graph in EBCDIC. The `main` branch stores everything in UTF-8 and has a cleaner API. New projects should prefer `main`.

## Build and Deploy (runs on IBM i via SSH)

All build commands run on the IBM i system, not locally. The repo is cloned to the IBM i IFS and compiled there.

```bash
# Full build — creates NOXDB library and JSONXML service program
gmake

# Full build with ISO 8601 timestamps as default
gmake ISO_TIMESTAMP=1

# Compile a single source file and update service program (VSCode workflow)
gmake current SRC=src/sqlio.c MODULE=SQLIO

# Clean compiled objects
gmake clean

# Clean and build release save file
gmake clean release
```

The build produces:
- `NOXDB/JSONXML` service program
- `NOXDB/QRPGLEREF.JSONPARSER`, `.XMLPARSER`, `.NOXDB`, `.JSONXML` — RPG prototype copybooks
- `NOXDB/NOXDB` binding directory

The `BIN_LIB` variable (default `NOXDB`) controls where objects land.

## Unit Tests (run on IBM i)

Requires [iRPGUnit](https://irpgunit.sourceforge.net) or [RPGUnit](https://rpgunit.sourceforge.net) to be installed on the IBM i.

```bash
cd unittests
# Compile unit tests
make RUINCDIR=/usr/local/include/irpgunit

# Custom library targets
make BIN_LIB=NOXDBUT NOXDB_LIB=NOXDB RU_LIB=IRPGUNIT
```

Before running tests, set the job's current directory on the IBM i: `CHGCURDIR '/path/to/noxDB'`

## Compile a Single Example or Test Program

```bash
# Example
gmake example SRC=json1

# Test program
gmake test SRC=issue123
```

## Architecture

### Core Data Flow

```
JSON/XML string or file
    ↓
jsonparser.c / xmlparser.c   (parse to node tree)
    ↓
noxdb.c                      (node tree API — navigate, create, modify)
    ↓
serializer.c / xmlserial.c   (serialize back to text)

SQL result sets → sqlio.c → node tree (same API)
```

### Key Source Files

| File | Role |
|------|------|
| `src/noxdb.c` | Core API: node tree creation, navigation, value get/set (~4000 LOC) |
| `src/sqlio.c` | SQL CLI integration: execute queries, fetch results into node tree (~3800 LOC) |
| `src/jsonparser.c` | JSON → node tree |
| `src/xmlparser.c` | XML → node tree |
| `src/serializer.c` | Node tree → JSON text |
| `src/xmlserial.c` | Node tree → XML text |
| `src/loadpgm.c` | Dynamic IBM i program binding (~1200 LOC) |
| `src/ext/mem001.c` | Custom memory manager with debug tracking |
| `src/ext/xlate.c` | EBCDIC/ASCII/UTF-8 character set conversion |

### Public API

Defined in `headers/JSONXML.rpgle` (RPG) and `headers/jsonxml.h` (C). Three binding variants are generated automatically from `JSONXML.rpgle`:
- `JSONPARSER.rpgle` — JSON-only API (prefix `json_`)
- `XMLPARSER.rpgle` — XML-only API (prefix `xml_`)
- `NOXDB.rpgle` — combined API

Core API patterns:
- **Parse**: `jx_parseFile()`, `jx_parseString()`, `jx_parseStringCcsid()`
- **Navigate**: `jx_locate()`, `jx_getChild()`, `jx_getNext()`, `jx_getParent()`
- **Modify**: `jx_setStr()`, `jx_setInt()`, `jx_setBool()`, `jx_setDate()`, `jx_setTimeStamp()`
- **Read**: `jx_getValueStr()`, `jx_getValueNum()`, `jx_getValueInt()`
- **Serialize**: `jx_asJsonText()`, `jx_asXmlText()`, `jx_writeJsonStmf()`, `jx_writeXmlStmf()`
- **SQL**: `jx_sqlFetch()`, `jx_sqlExecute()`, `jx_sqlDescribe()`
- **Lifecycle**: `jx_delete()`, `jx_error()`, `jx_message()`

Node navigation uses XPath-like path strings, e.g. `/customer/address[0]/street`.

### IBM i Specifics

- All C source files must have CCSID 1252 (WIN-1252) — the makefile sets this via `CHGATR` before compiling
- `TERASPACE(*YES)` enables 64-bit memory addressing
- `ACTGRP(QILE)` — service program runs in the QILE activation group
- `TARGET_RLS=*PRV` — builds for one release back for compatibility
- The VSCode `.vscode/tasks.json` configures remote IBM i compile tasks using Code for IBM i extension
