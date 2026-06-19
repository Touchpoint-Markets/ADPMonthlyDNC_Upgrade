# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **SQL Server Integration Services (SSIS)** project (`ADPMonthlyDNC_Upgrade`) that automates monthly ADP Do-Not-Contact (DNC) list processing for the Diamond360 database. It downloads suppression CSVs via WinSCP, validates them, loads them into SQL Server DNC tables, runs a stored procedure export, archives the files, and sends email notifications.

## Build and Deploy

The project is developed and built using **SQL Server Data Tools (SSDT) in Visual Studio 2022** (product version 17.0.1016.0). There is no CLI build command — all operations go through the SSDT IDE.

- **Open solution**: `ADPMonthlyDNC_Upgrade.slnx`
- **Build configuration**: `Development` (the only configuration defined)
- **Build output**: `bin\Development\ADPMonthlyDNC.ispac` — this `.ispac` file is the deployment artifact for SSIS Catalog deployment
- **Target server version**: SQL Server 2025

To deploy: use the SSIS Deployment Wizard (`dtutil` or SSISDB Catalog) with the built `.ispac` file.

## Package Control Flow

`ADPMonthlyDNC_Upgrade.dtsx` is the only package in the project. The execution order (defined via `DTS:PrecedenceConstraints`) is:

```
Execute Process Task (WinSCP download)
    └─► Check Files Downloaded or Not
            ├─► [FAILURE] File not found Email
            └─► [SUCCESS] Foreach Loop Container (iterates *.csv in SourceFolder)
                    └─► Get Path of Files (routes to EmailsFilePath / PhonesFilePath)
                └─► Truncate DNC Tables
                        └─► Populating DNC Tables (Data Flow)
                                └─► Export Process (EXEC MonthlyJDAUpdateDNCTables)
                                        └─► Move Files to Archive
                                                └─► Final Confirmation Email
```

**Task summary:**

| Task | Type | Details |
|------|------|---------|
| Execute Process Task | Execute Process | Runs `WinSCP.exe`; executable and arguments driven by `User::WinSCPExecutablePath`, `User::WinSCPScriptPath`, `User::WinSCPLogPath` via property expressions |
| Check Files Downloaded or Not | C# Script Task | Fails if fewer than 2 `.csv` files exist in `SourceFolder` |
| File not found Email | C# Script Task | Sends SMTP notification email on file-not-found failure |
| Foreach Loop Container | Foreach File Enumerator | Iterates `*.csv` in `User::SourceFolder`; maps each path to `User::CurrentFile` |
| Get Path of Files | C# Script Task | Sets `User::EmailsFilePath` or `User::PhonesFilePath` based on filename containing "emails" or "phones" |
| Truncate DNC Tables | Execute SQL | `TRUNCATE TABLE JDContactsDNC; TRUNCATE TABLE JDContactsPhoneDNC;` |
| Populating DNC Tables | Data Flow Task | Loads EmailsCSV → `JDContactsDNC`; PhoneCSV → `JDContactsPhoneDNC` |
| Export Process | Execute SQL | `EXEC Diamond360.dbo.MonthlyJDAUpdateDNCTables` |
| Move Files to Archive | C# Script Task | Moves all files from `SourceFolder` to `ArchiveFolder`; skips if already exists |
| Final Confirmation Email | Script Task | Queries DB via `EmailConnection` for sample records, sends success email |

Package-level `OnError` event handler fires `Send Mail on Error` on any task failure.

## Connection Managers

| Name | Type | Target |
|------|------|--------|
| `DBConnection` | OLE DB (MSOLEDBSQL.1) | `10.9.57.8`, `Diamond360`, user `jdauser` |
| `EmailConnection` | ADO.NET (SqlClient) | Same server/database (`10.9.57.8`) — used in Final Confirmation Email to query sample records |
| `EmailsCSV` | Flat File | Path set dynamically from `User::EmailsFilePath` |
| `PhoneCSV` | Flat File | Path set dynamically from `User::PhonesFilePath` |

## Key Variables

| Variable | Default Value |
|----------|---------------|
| `User::SourceFolder` | `I:\...\Downloaded Files` |
| `User::ArchiveFolder` | `I:\...\Archive` |
| `User::CurrentFile` | Set by Foreach enumerator each iteration |
| `User::EmailsFilePath` | Set by Get Path of Files script; used as `EmailsCSV` connection string |
| `User::PhonesFilePath` | Set by Get Path of Files script; used as `PhoneCSV` connection string |
| `User::WinSCPExecutablePath` | `C:\Program Files (x86)\WinSCP\WinSCP.exe` |
| `User::WinSCPScriptPath` | `\\CINPSQL20\ADP Auto Process Dont change folder data\Command.txt` |
| `User::WinSCPLogPath` | `\\CINPSQL20\ADP Auto Process Dont change folder data\winscp_log.txt` |
| `User::SMTPServer` | `email-smtp.us-east-1.amazonaws.com` |
| `User::SMTPPort` | `587` (Int32) |
| `User::SMTPUsername` | AWS SES access key |
| `User::SMTPPassword` | AWS SES secret key |
| `User::EmailFrom` | `ADPAutoProcess@judydiamond.com` |
| `User::EmailTo` | `eric.ryles@arc-network.com` |
| `User::EmailCC` | Synoptek + arc-network recipients |
| `User::ErrorEmailFrom` | `ErrorOnADPAutoProcess@judydiamond.com` |
| `User::ErrorEmailTo` | `hvaghasiya@synoptek.com` |

## Protection Level

The package uses `EncryptSensitiveWithUserKey`. Opening the package on a different machine or as a different Windows user will require re-entering sensitive connection credentials.

## File Naming Convention

The `Get Path of Files` script routes files to the appropriate flat-file connection manager based on the filename:
- Filename containing **"emails"** → `User::EmailsFilePath` (loaded into `JDContactsDNC`)
- Filename containing **"phones"** → `User::PhonesFilePath` (loaded into `JDContactsPhoneDNC`)

Both CSV files must be present (≥2 files total in `SourceFolder`) for processing to proceed.

## External Dependencies

- **WinSCP** installed at `User::WinSCPExecutablePath` on the SSIS execution host
- WinSCP command script path configured in `User::WinSCPScriptPath`
- SQL Server `10.9.57.8` with `Diamond360` database and stored procedure `dbo.MonthlyJDAUpdateDNCTables`
- Network share `I:\` mapped on the SSIS host (points to `\\CINPSQL20\ADP Auto Process Dont change folder data\`)
- AWS SES SMTP endpoint configured in `User::SMTPServer` / `User::SMTPPort` / `User::SMTPUsername` / `User::SMTPPassword`
