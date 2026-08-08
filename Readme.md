.............Go_projects\piza_o_kudasai> go mod init pizza-tracker-go
go: creating new go.mod: module pizza-tracker-go

mkdir internal/models, templates/static, data, cmd


https://gorm.io/

https://github.com/teris-io/shortid

run :  go mod tidy

if : 
1. Enable cgo

In PowerShell:

go env -w CGO_ENABLED=1

Check:

go env CGO_ENABLED

Expected:

1
2. Run your project

go run ./cmd  
cntr + c 