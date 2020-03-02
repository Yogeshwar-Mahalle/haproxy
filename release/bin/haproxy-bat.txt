@echo off
rem #####################################################################################
rem # Description: 
rem #   HA-Proxy is a TCP/HTTP reverse proxy for high availability and load balancing.
rem # Author: Yogeshwar Mahalle 
rem # Version: 2.1.3
rem #####################################################################################

set BASENAME=haproxy
set HAPROXY-HOME=%~dp0
set BIN=%BASENAME%-2.1.3-win64.exe
set CFG=../config/haproxy.cfg
set PIDFILE=haproxy.pid
set LOCKFILE=haproxy.lock

set CFG=%HAPROXY-HOME%%CFG%
set PIDFILE=%HAPROXY-HOME%%PIDFILE%
set LOCKFILE=%HAPROXY-HOME%%LOCKFILE%

echo Loading %BASENAME% ....

if "%1" == "start" (
  call :start
) else if "%1" == "stop" (
  call :stop
) else if "%1" == "restart" (
  call :restart
) else if "%1" == "reload" (
  call :reload
) else if "%1" == "condrestart" (
  call :condrestart
) else if "%1" == "status" (
  call :rhstatus
) else if "%1" == "check" (
  call :check
) else (
  echo "Usage: %BASENAME% {start|stop|restart|reload|condrestart|status|check}"
)
goto eof
  
if not exist "%CFG%" (
  echo Configuration not found
  goto :eof
) 


:start

  if exist "%LOCKFILE%" (
    echo Already Running. Please stop the running process and then start.
    goto :eof
  )

  call :quiet_check
  if not ERRORLEVEL 0 (
    echo Errors found in configuration file, check it with '%BASENAME% check'
    goto :eof
  )

  echo Starting %BASENAME% .... 
  %BIN% -D -f "%CFG%" -p "%PIDFILE%"
  IF errorlevel 0  type nul > "%LOCKFILE%"
  
  for /f "tokens=2" %%P IN ('TASKLIST /NH /FI "imagename eq %BIN%" /svc' ) Do (set PID=%%P)
  echo %PID% > "%PIDFILE%"
  echo %BASENAME% started with PID : %PID%
  EXIT /B 0



:stop
  echo Shutting down %BASENAME% ....

  for /f "usebackq delims=" %%l in (%PIDFILE%) do (
    echo Stopping process %%l
    taskkill /F /PID %%l
    set RETVAL=%errorlevel%
    echo Stopping status %RETVAL%
  )

  tasklist /fi "imagename eq %BIN%" |find ":" > nul
  if errorlevel 1 taskkill /f /im "%BIN%"
  set RETVAL=%errorlevel%

  if %RETVAL% EQU 0 ( del "%LOCKFILE%" )
  if %RETVAL% EQU 0 ( del "%PIDFILE%" )
  EXIT /B 0



:quiet_check
  %BIN% -c -q -f %CFG%
  EXIT /B 0

:restart
  
  call :quiet_check
  if not errorlevel 0 (
    echo Errors found in configuration file, check it with '%BASENAME% check'
    goto :eof
  )
  
  call :stop
  call :start
  EXIT /B 0


:reload
  if not exist "%PIDFILE%" (
    goto :eof
  )

  call :quiet_check
  if not errorlevel 0  (
    echo Errors found in configuration file, check it with '%BASENAME% check'
    goto :eof
  )
  
  set PIDS=
  for /f "usebackq delims=" %%i in (%PIDFILE%) do set PIDS=%PIDS% %%i
  %BIN% -D -f "%CFG%" -p "%PIDFILE%" -sf %PIDS%
  EXIT /B 0


:check
  %BIN% -c -q -V -f "%CFG%"
  EXIT /B 0


:rhstatus
  TASKLIST /FI "imagename eq %BIN%" /svc
  FOR /F %%x IN ('tasklist /NH /FI "IMAGENAME eq %BIN%"') DO IF %%x == %BIN% goto FOUND
  echo The %BASENAME% is not running
  goto NOTFOUND
:FOUND
  echo The %BASENAME% is running
:NOTFOUND
  EXIT /B 0


:condrestart
  if exist "%LOCKFILE%" (
    call :restart
  )
  
  EXIT /B 0




:eof
EXIT /B 1
