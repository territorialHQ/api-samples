# Bot Endpoint Documentation

TerritorialHQ interacts with various discord bots to display points earned by users and to offer a seamless and automatic management of points for the end user. To support this, your bot will need to implement a simple endpoint that allows to interact with the TerritorialHQ Infrastructure. <br>
**After setting up the endpoint make sure to add it to your clan at "Custom Bot HttpGet Endpoint"** <br>
**If you want to support automatic point submission, dm @platz1de on discord with your endpoint url and information about your use-case**

## GET Endpoint

The endpoint has to support this protocol to be accepted by TerritorialHQ:

```http
GET /:user HTTP/1.1
Host: example.com

HTTP/1.1 200 OK
Content-Type: text/plain

HTTP/1.1 404 Not Found
```

Where `:user` is the discord user id of the user that is requesting the points. <br>
The endpoint has to return the amount of points the user has as a plain text response. <br>
If the user is not found, the endpoint has to return a 404 response.

### Example

Here is an example of a simple endpoint that returns the amount of points a user has:

```javascript
import {createServer} from 'http';
import sqlite3 from 'sqlite3';

const db = new sqlite3.Database(/*...*/);

createServer(function (r, s) {
	if (r.url.startsWith("/user/")) {
		let username = r.url.substr(6);
		db.get("SELECT points FROM table WHERE member = ?", [username], (err, row) => {
			if (!row || err) {
				s.writeHead(404);
				s.write("User not found");
			} else {
				s.write(row.points.toString());
			}
			s.end();
		});
	}
}).listen(8080);
```

**Important Notice:** You are dealing with unsanitized user input. Make sure your code is secure and does not allow for SQL injections or other attacks.

## POST Endpoint

The post endpoint is used to submit results of games. All results are verified by TerritorialHQ and only accepted and sent if they are valid. <br>
This protocol is optional and only required if you want to support automatic point submission. <br>

```http
POST /:user HTTP/1.1
Host: example.com
Content-Type: text/plain
```

Example payload
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IjAwMDAwMDAwMDAwMDAwMDAwMCIsInBvaW50cyI6NTAsImNsYW4iOiIwMDAwMDAwMDAwMDAwMDAwMDAiLCJpYXQiOjE2ODkxMjAwNzQsIm5iZiI6MTY4OTEyMDA3NCwiZXhwIjoxNjg5MTIwMTM0fQ.PJJZQR_vZS93iZnxovKwKo3ps0x1wgEi_KhbsnIkLrs5E3CPn28ymaa8GE8QZtbxT490nwzRxTPHLCs0Ng1pjb9dSLQWX2_4ukJcww6Guftm1K65bqDpxIO1HEHLmmKYXcd4LJOYeWJVQbvgJisg5Yx_LLeA6eNgHw6yDJfzp45-y__ccrgGjtUw-9xrythy1zs4nf4YIX-4-8rbo-BVG3fcz_TqMs77xnqsN3UHvEAv_YDvdvQ_5krgbq-OZIblcbNlBVxBzHUoMT-KvwTFe6ITGwyEQxiF46aYnbqPB9m_iaGOv1BSndaS6ciCgRUBZJ9s7cBhqtjeU5bE6CchjQ
```

Where `:user` is the discord user id of the user the points should be added to. <br>
The request body is a JWT token containing the following information:

```json
{
  "username": "000000000000000000",
  "points": 50,
  "clan": "3e39fe4d-df0a-46d0-b99e-6a4144569984",
  "iat": 1689120074,
  "nbf": 1689120074,
  "exp": 1689120134
}
```

The token can be verified using the following public key. <br>
Make sure to also check whether the username matches the user id in the url and that the clan field matches your clans discord guild id. <br>
Depending on your implementation you might also want to check the `iat`, `nbf` and `exp` fields to make sure the token is within the valid life span.

```
-----BEGIN RSA PUBLIC KEY-----
MIIBCgKCAQEAgrtsxtwTViEldst5pNxtCS65aPwiwhJjs2785zmqq6iK10Sr2/cT
mPJSdnE3x9VliGctUakxFOyCe5b+bo3BdVKuyjsX2UvVWCFRiSQBwOOEGX0eqLui
9Yvor8/QF3TbNJaQfT5f2o9ZWtQN6zCcP/MJcZ9l1Rl+XfsZQkOgkDnb/q9Qpp0L
7+c+2gpV34J+sA/7QNirQGHZzzXbJrqoROywQAPyN/a7UtmPmciRAgYWLOvJkYmM
FcnhTDc+uwDj7C1giSXZBTX8HsIclv+sZjKxkdLN3juCUc0zbYzX3e4Vl+vn2zKZ
SsLbtIgZ4C6iuDooeGmoZC6HVmuUNfCoJQIDAQAB
-----END RSA PUBLIC KEY-----
```

### Example

Here is an example of a simple endpoint that accepts the token and adds the points to the user:

```javascript
import {createServer} from 'http';
import sqlite3 from 'sqlite3';
import jwt from 'jsonwebtoken';

const db = new sqlite3.Database(/*...*/);
const publicKey = `PUBLIC KEY (see above)`;
const myClanId = "DISCORD SERVER ID HERE";

createServer(function (r, s) {
    if (r.url.startsWith("/user/") && r.method === "POST") {
        let username = r.url.substr(6);
        let body = "";
        r.on("data", (chunk) => {
            body += chunk;
        });
        r.on("end", () => {
            let token = body;
            let decoded = jwt.verify(token, publicKey);
            if (decoded.username !== username || decoded.clan !== myClanId) {
                s.writeHead(400);
                s.write("Bad Request");
                s.end();
                return;
            }
            db.run("UPDATE table SET points = points + ? WHERE member = ?", [decoded.points, decoded.username], (err) => {
                if (err) {
                    s.writeHead(500);
                    s.write("Internal Server Error");
                } else {
                    // It's recommended to send some kind of log to your clan's discord server here
                    s.writeHead(200);
                    s.write("OK");
                }
                s.end();
            });
        });
    }
}).listen(8080);
```