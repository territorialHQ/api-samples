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
POST / HTTP/1.1
Host: example.com
Content-Type: text/plain
```

Example payload
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbGllbnRzIjpbeyJ1c2VybmFtZSI6IjExMDc0MTU5MTQ4NjI4Nzg4NzEiLCJwb3NpdGlvbiI6MX0seyJ1c2VybmFtZSI6IjExMjQ2OTEzMTE2NjA4OTIyNDIiLCJwb3NpdGlvbiI6Nzh9XSwidGVhbUNvdW50Ijo0LCJwb2ludHMiOjQyMCwibWFwIjoxMCwiY2xhbiI6IjExMDE5ODE2MTEyNDQ5MjUwMTAiLCJpYXQiOjE2ODkxMjAwNzQsIm5iZiI6MTY4OTEyMDA3NCwiZXhwIjoxNjg5MTIwMTM0fQ.aJE1zcJyPY0nr4fp_GV2N9MIcQpnSzM_b-MjkF5cHFzAyt2hWM5Er3kn0pH4vOG7xoKwZQFaYnYqCXJMPL_DN5Apl7jKjKstOMQr1R3yWhAxTDDnJ1NjASpvmm9b0bAYERujb0XF_ZSEYBO8WdLbQJ5uLxysq-ULnn7lv7RJrRRzaeLCiLX_I8Vcfxzduw1i4xIkR3qY1lVMKRhqSmhi_fXon3TEX4gA82OSnDtxVRc1xFTkoUV3TTYJmCzG8FuWGDToqc9JYA_httFFG6U2KlWMsZ9Wlz121jhDsT6oD6LN3IMSTrzPH3avMSE_Ro0BXIlExfhKU44lKOq4Z4CXiQ
```

The request body is a JWT token containing the following information:

```json
{
  "clients": [
	{
	  "username": "1107415914862878871",
	  "position": 1
	},
	{
	  "username": "1124691311660892242",
	  "position": 78
	}
  ],
  "teamCount": 4,
  "points": 420,
  "map": 10,
  "clan": "1101981611244925010",
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
	if (r.method === "POST") {
		let body = "";
		r.on("data", (chunk) => {
			body += chunk;
		});
		r.on("end", () => {
			let token = body;
			let decoded = jwt.verify(token, publicKey);
			if (decoded.clan !== myClanId) {
				s.writeHead(400);
				s.write("Bad Request");
				s.end();
				return;
			}
			for (let client of decoded.clients) {
				// You can implement any kind of logic related to player positions, map type, etc. here
				db.run("UPDATE table SET points = points + ? WHERE member = ?", [decoded.points, client.username], (err) => {
					// It's recommended to send some kind of log to your clan's discord server here
				});
			}
			s.writeHead(200);
			s.write("OK");
			s.end();
		});
	}
}).listen(8080);
```