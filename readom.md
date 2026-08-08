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

sqlite3 -header -column data/orders.db "SELECT * FROM orders;"

sqlite3 data/orders.bd "UPDATE orders SET status = 'Preparing' WHERE id = 'ArzL3bsvg';" 


PS D:\webProjects\Gaylang\Go_projects\piza_o_kudasai> sqlite3 data/orders.db "DELETE FROM users WHERE username = 'admin';"
PS D:\webProjects\Gaylang\Go_projects\piza_o_kudasai> @'
>> INSERT INTO users (username, password)
>> VALUES ('admin', '$2a$12$2NhprrFLakHKbJYiaTzcM.ntQdFWkCY.Kbm4bahLqilUmfjvCqSha');
>> '@ | sqlite3 data/orders.db

PS D:\webProjects\Gaylang\Go_projects\piza_o_kudasai> sqlite3 -header -column data/orders.db "SELECT id, username, password FROM users;"
id  username  password                                                    
--  --------  ------------------------------------------------------------
2   admin     $2a$12$2NhprrFLakHKbJYiaTzcM.ntQdFWkCY.Kbm4bahLqilUmfjvCqSha

PS D:\webProjects\Gaylang\Go_projects\piza_o_kudasai> 

password1234