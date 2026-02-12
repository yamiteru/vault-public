App for slow communication. Main goal is for people to not be addicted to instant messaging.

## Tech stack
### Frontend
- Server = Cloudflare Pages
- Framework = Solid + Styled components

### Backend
- Server = Cloudflare Workers
- API = Custom tRPC-inspired with binary protocol
- Database = CockroachDB

### Shared
- Data validation = Custom Zod-inspired data validation
- Error logs = Rollbar

## Database schema
- Person
	- id: string (uuid v4)
	- avatar: string (S3 asset id)
	- name: string
	- nick[index]: string
	- email[index]: string
	- password: string (argon2 hashed)
	- dateRegistered: date
	- allowed: Person[]
	- blocked: Person[]
	- pending: Person[]
	- inbox: Letter[]
- Letter
	- id: string (uuid v4)
	- from: Person
	- title: string
	- content: string
	- media: string[] (S3 asset id array)
	- dateCreated: date