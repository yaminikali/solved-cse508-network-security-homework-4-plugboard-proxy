Download Link: https://assignmentchef.com/product/solved-cse508-network-security-homework-4-plugboard-proxy
<br>
In this assignment you will develop a “plugboard” proxy for adding an extra

layer of protection to publicly accessible network services. Your program will

be written in Go using the Crypto library.




Consider for example the case of an SSH server with a public IP address. No

matter how securely the server has been configured and how strong the keys

used are, it might suffer from a “pre-auth” zero day vulnerability that allows

remote code execution even before the completion of the authentication

process. This could allow attackers to compromise the server even without

providing proper authentication credentials. The Heartbleed OpenSSL bug is an

example of such a serious vulnerability against SSL/TLS.




The plugboard proxy you are going to develop, named ‘pbproxy’, adds an extra

layer of encryption to connections towards TCP services. Instead of connecting

directly to the service, clients connect to pbproxy (running on the same

server), which then relays all traffic to the actual service. Before relaying

the traffic, pbproxy *always* decrypts it using a static symmetric key. This

means that if the data of any connection towards the protected server is not

properly encrypted, then it will turn into garbage before reaching the

protected service.




This is a better option than port knocking and similar solutions, as attackers

who might want to exploit a zero day vulnerability in the protected service

would first have to know the secret key for having a chance to successfully

deliver their attack vector to the server. This of course assumes that the

plugboard proxy does not suffer from any vulnerability itself. Given that its

task and its code are much simpler compared to an actual service (e.g., an SSH

server), its code can be audited more easily and it can be more confidently

exposed as a publicly accessible service. Go is also a memory-safe language

that does not suffer from memory corruption bugs.




Clients who want to access the protected server should proxy their traffic

through a local instance of pbroxy, which will encrypt the traffic using the

same symmetric key used by the server. In essence, pbproxy can act both as

a client-side proxy and as server-side reverse proxy, in a way similar to

netcat.




Your program should conform to the following specification:




go run pbproxy.go [-l listenport] -p pwdfile destination port




-l  Reverse-proxy mode: listen for inbound connections on &lt;listenport&gt; and

relay them to &lt;destination&gt;:&lt;port&gt;




-p  Use the ASCII text passphrase contained in &lt;pwdfile&gt;




* In client mode, pbproxy reads plaintext traffic from stdin and transmits it

in encrypted form to &lt;destination&gt;:&lt;port&gt;




* In reverse-proxy mode, pbproxy should continue listening for incoming

connections after a previous session is terminated, and it should be able to

handle multiple concurrent connections (all using the same key).




* Data should be encrypted/decrypted using AES-256 in GCM mode (bi-directional

communication). You should derive an appropriate AES key from the supplied

passphrase using PBKDF2.




Going back to the SSH example, let’s see how pbproxy can be used to protect an

SSH server. Assume that we want to protect a publicly accessible sshd running

on vuln.cs.stonybrook.edu. First, we should configure sshd to listen *only* on

the localhost interface, making it inaccessible from the public network. Then,

we fire up a reverse pbproxy instance on the same host listening on port 2222:




pbproxy -p mykey -l 2222 localhost 22




Clients can then connect to the SSH server using the following command:




ssh -o “ProxyCommand pbproxy -p mykey vuln.cs.stonybrook.edu 2222” localhost




This will result in the following data flow:




ssh &lt;–stdin/stdout–&gt; pbproxy-c &lt;–socket 1–&gt; pbproxy-s &lt;–socket 2–&gt; sshd

______________________________/                ___________________________/

client                                        server




Socket 1 (encrypted): client:randomport &lt;-&gt; server:2222

Socket 2 (plaintext): localhost:randomport &lt;-&gt; localhost:22




To test your setup, you can achieve a similar data flow using netcat instead

of pbproxy, by first running it on the same server as sshd as follows:




nc -l -p 2222 -c ‘nc localhost 22’




Then connecting from the client machine as follows:




ssh -o “ProxyCommand nc vuln.cs.stonybrook.edu 2222” localhost

Hints:




1) Mind your Nonces!




2) Some useful resources:




https://cryptobook.nakov.com/symmetric-key-ciphers/cipher-block-modes

https://pkg.go.dev/net

https://pkg.go.dev/crypto/cipher

https://pkg.go.dev/golang.org/x/crypto/pbkdf2