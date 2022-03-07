---
title: How to protect login API against brute force attack by limiting login attempts
categories:
- Nodejs
- RESTful API
- rate-limiter-flexible
---
<div style="position: relative; padding-bottom: 65.29516994633273%; height: 0;"><iframe src="https://www.loom.com/embed/eb8f525571dd484fb8b6492a61f81213" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe></div>
<p></p>
SECURITY is at the heart of REST API design best practices because the vulnerability of an endpoint if not well secured can be exploited by an attacker to cause serious damage to the system or gain access to sensitive information using illegitimate credentials. In this article, we’ll look at how to design a Login REST API that is easy to understand for anyone and consume, well secured against various forms of attacks, and can be integrated into a new or existing system for proper user authentication.

Before we proceed to implementing the Login API, it is however very important to look at some of the common vulnerability of RESTful API's, attacks and how to prevent them following REST API design common convetion.


# DDoS attack
A distributed denial-of-service (DDoS) attack occurs when there's a deliberate attempt to ruin the normal traffic of a server, network, or services by overwhelming the server with a flood of requests. This is arguably one of the most common forms of API attacks which are carried out with the sole purpose of preventing access to a targeted server or service.


# How to prevent DDoS attack
The best way to prevent this kind of attack is to implement rate limiting on your server infrastructure. Rate limiting is a strategy for controlling the rate of requests sent or received by a network or server. It puts a cap on the number of requests someone can send to a targeted server within a certain timeframe. 
<!-- more -->

# Brute force attack
A brute force attack, also known as brute force cracking is a common type of system authentication vulnerability exploitation by a cyberattacker. A brute force attacker uses trial and error in an attempt to guess or crack an account password, user login credentials, and encryption keys.


# How to prevent brute force attack
The best and most common prevention strategy against brute force attacks is by limiting login attempts and lockout the targeted account after a certain number of login attempts. Implementing strong password input validation policy, setting up mechanisms for two-factor authentication, restricting or blacklisting malicious IP addresses, and using CAPTCHAs.


# SQL Injection Attacks
SQL Injection Attack is a trick that attackers use to trick the server to gain authorization illegally and access the database by using SQL queries. The objective of this kind of attack is to break into a database infrastructure using malicious SQL queries to manipulate the server thereby gaining access to information that is not meant to be displayed. This information may include credit card information, company sensitive data, client personal information, etc.


# How to prevent SQL Injection Attacks
The common and most effective method of preventing a SQL Injection Attack is proper input validation. Mostly, SQL statements for retrieving data from the database dynamically may require some query params which come as input from the client. Therefore, the server should never allow or accept direct param input without proper sanitization. There are however so many input validation methods and techniques that can be adopted to achieve this purpose depending on the complexity of the system design.


# Cross-site scripting (XSS)
 Cross-site scripting is similar to SQL Injection Attacks but in this case, the attacker exploits the vulnerability of the system (application) also known as a loophole to insert a malicious script (often JavaScript) into the code of a web app or webpage


# How to prevent Cross-site scripting (XSS)
Just like in SQL Injection Attacks prevention, it is also very important to scrutinize and validate all inputs before sending them to the backend server.

These of course are some of the most common REST API vulnerability and attack methods that we need to take into cognizance when designing a new RESTful API or when refactoring. It is therefore very important to put adequate measures in place such that the API can withstand a test of time against any attack.

# Login API with login attempt limmiting strategy in nodejs 
We are going to design a login API using nodejs, the endpoint will be secured by rate-limiting and login attempts limiting strategy. It will help prevent brute-force attacks and DDoS attacks. There are many nodejs packages at your disposal that can be used to achieve the said objective but of course, whichever one you choose to use depends on the kind of project you're working on, the requirements, and complexity.

For the purpose of this tutorial, we'll make use of * *rate-limiter-flexible* nodejs package for handling rate-limiting [official npm website](https://www.npmjs.com/package/rate-limiter-flexible) and *redis* package. rate-limiter-flexible uses Redis cache for storing requests and login attempts.

First, let's create a helper function that will be responsible for handling rate-limiting and catching login attempts

```js
// Lets start by defining dependencies 
const { RateLimiterRedis } = require('rate-limiter-flexible');
const { redis_conn } = require('../../../config/cache');

// Protect login route agains brute force and too many login attempts 
const loginAttemptLimitter = async (email, ip,) => {
    return new Promise(async function (resolve, reject) {
        // set maximum wrong attempts to 100 by an ip address within 24hrs
        const maxWrongAttemptsByIPperDay = 100;   
        // set maximum wrong attemps to 5 by combination of username and ip address in minutes
        const maxConsecutiveFailsByUsernameAndIP = 5;
        // set maximum wrong attempts to 50 by username per day   
        const maxWrongAttemptsByUsernamePerDay = 50;  

        // creates Redis catche for storing request attempt from an IP address
        const limiterSlowBruteByIP = new RateLimiterRedis({
            redis: redis_conn,
            keyPrefix: 'login_fail_ip_per_day',
            points: maxWrongAttemptsByIPperDay,
            duration: 60 * 60 * 24,
            blockDuration: 60 * 60 * 24, // Block for 1 day, if 100 wrong attempts per day
        });

        // creates Redis catche for storing login attempt by a username and IP address
        const limiterConsecutiveFailsByUsernameAndIP = new RateLimiterRedis({
            redis: redis_conn,
            keyPrefix: 'login_fail_consecutive_username_and_ip',
            points: maxConsecutiveFailsByUsernameAndIP,
            duration: 60 * 60 * 24 * 90, // Store number for 90 days since first fail
            blockDuration: 60 * 60, // Block for 1 hour
        });

        // creates Redis catche for storing login attempt by username 
        const limiterSlowBruteByUsername = new RateLimiterRedis({
            redis: redis_conn,
            keyPrefix: 'login_fail_username_per_day',
            points: maxWrongAttemptsByUsernamePerDay,
            duration: 60 * 60 * 24,
            blockDuration: 60 * 60, // Block for 1 hour
        });


        const usernameIPkey = (email, ip) => `${email}_${ip}`;
        // returns the content of limmiters stored in the catche 
        const [resUsernameAndIP, resSlowByIP, resSlowUsername] = await Promise.all([
            limiterConsecutiveFailsByUsernameAndIP.get(usernameIPkey),
            limiterSlowBruteByIP.get(ip),
            limiterSlowBruteByUsername.get(email),
        ]);
        let retrySecs = 0;

         if (resSlowByIP !== null && resSlowByIP.consumedPoints > maxWrongAttemptsByIPperDay) {
            retrySecs = Math.round(resSlowByIP.msBeforeNext / 1000) || 1;
            retrySecs = parseInt(Math.floor(retrySecs / 60))
            console.error({
                status: 'blocked',
                error: `Too Many Requests from ip--${ip} in a short time retry after ${retrySecs} minutes`
            });
            return reject({
                status: 'blocked',
                error: `Too Many Requests from ip--${ip} in a short time retry after ${retrySecs} minutes`
            });
        } else if (resUsernameAndIP !== null && resUsernameAndIP.consumedPoints > maxConsecutiveFailsByUsernameAndIP) {
            retrySecs = Math.round(resUsernameAndIP.msBeforeNext / 1000) || 1;
            console.error({
                status: 'blocked',
                message: `Too Many login attempts from username--${email} & ip--${ip} retry after ${retrySecs} minutes`
            });
            return reject({
                status: 'blocked',
                message: `Too Many login attempts from username--${email} & ip--${ip} retry after ${retrySecs} minutes`
            });
        } else if (resSlowUsername !== null && resSlowUsername.consumedPoints > maxWrongAttemptsByUsernamePerDay) {
            retrySecs = Math.round(resSlowUsername.msBeforeNext / 1000) || 1;
            console.error({
                status: 'blocked',
                message: `Too Many login attempts from username--${email} retry after ${retrySecs} minutes`
            });
            return reject({
                status: 'blocked',
                message: `Too Many login attempts from username--${email} retry after ${retrySecs} minutes`
            });
        } else {
            return resolve({ limiterSlowBruteByIP, limiterConsecutiveFailsByUsernameAndIP, limiterSlowBruteByUsername, usernameIPkey });
        }
    });
};
```

Next, we'll create the Login API and secure it using the helper function created above. But before we delve into designing the login endpoint, it is assumed that you already have a secure and working database setup. Setting up a database is beyond the scope of this tutorial.

At a minimum, an authentication table must have at least an email, password, and isVerified column for user validation
<table>
    <tr>
        <td>Email</td>
        <td>Password</td>
        <td>isVerified</td>
    </tr>
</table>

```js
const login = async (req) => {
    return new Promise(async (resolve, reject) => {
        const email = req.body.email;
        const ipAddress = req.connection.remoteAddress; //extract User IP address from req onbject
        const password = req.body.password;
        //check login attempts using our helper function 
        return await loginAttemptLimitter(email, ipAddress)
            .then(async (loginLimitters) => {
                const usernameIPkey = loginLimitters.usernameIPkey;
                // extract limmiters 
                const {
                    limiterSlowBruteByIP,
                    limiterConsecutiveFailsByUsernameAndIP,
                    limiterSlowBruteByUsername,
                } = loginLimitters;

                //Query your database using the suplied email for user validation and existence 
                const userData = await userModel.getUser(email);
                
                // My database returns null if the said user is not found
                // Your database response might be diffrent from mine 
                // So you might need to handle the response diffrently
                if (!userData) {
                    // Consume 1 point from limiters on wrong attempt by non existing user
                    limiterSlowBruteByIP.consume(ipAddress);
                    // Block if attempt limmit is exhausted
                    await limiterSlowBruteByIP;
                    console.log({
                        status: false,
                        message: 'user does not exist ',
                    })
                    return reject({
                        status: false,
                        message: 'user does not exist ',
                    });
                }

                // If user exists and not verified 
                if (!userData.is_verified) {
                    // Store failed attempts by Username only for unverified registered users
                    // Consume 1 point from limiters on wrong attempt by unverified users
                    limiterSlowBruteByUsername.consume(email);
                    // Block if attempt limmit is exhausted
                    await limiterPromises;
                    console.log({
                        status: false,
                        message: 'user exist but not verified ',
                    })
                    return reject(
                        {
                            status: false,
                            message: 'user exist but not verified ',
                        }
                    );
                }

                /** 
                 If the user exists and is verified compare the supplied password against the password in the database.
                 Mind you please, what I did here is not enough validation check for a password you can use a more secure mechanism
                 like JWT and Passport. but for the purpose of this tutorial I'm trying to keep things simple
                */
                if (password !== userData.password) {
                    // Count failed attempts by Username + IP only for verified registered users
                    // Consume 1 point from limiters on wrong attempt by verified users
                    limiterConsecutiveFailsByUsernameAndIP.consume(usernameIPkey);
                    // Block if attempt limmit is exhausted
                    await limiterPromises;
                    console.log({ message: 'Invalid password' })
                    return reject({ message: 'Invalid password' });
                }

                /** 
                If the user is successfully validated and has not exhausted the login attempt limit,
                reset caught login attempt by username and IP to zero
                */
                if (limiterConsecutiveFailsByUsernameAndIP !== null && limiterConsecutiveFailsByUsernameAndIP.consumedPoints > 0) {
                    // Reset counters after successful authorisation
                    await Promise.all(
                        limiterSlowBruteByIP.delete(),
                        limiterSlowBruteByUsername.delete(),
                        limiterConsecutiveFailsByUsernameAndIP.delete(usernameIPkey),
                    );

                }

                // user logged in 
                console.log({
                    status: true,
                    message: 'User logged in successfully'
                })

                return resolve({
                    status: true,
                    message: 'User logged in successfully'
                });
            })
            .catch((error) => {
                return reject(error)
            });
    });
```

lets create a controller function for handling login req
```js
const executeLogin = async (req, res) => {
    return await login(req)
        .then((obj) => {
            res.status(200)
            res.send(obj)
        })
        .catch(error => {
            res.status(400)
            res.send(error)
        })
}
```

Finally let's create the login endpoin.
```js
// Login API 
module.exports = (server) => {
    server.post({
        path: '/login',
    },
        (req, res) => {
            return executeLogin(req, res)
        }
    )
}
```

And that's ii guys we are done but of course, there may be better ways to do this. The idea here is to give you a good example of how to implement a rate limiter and login limiter that can be easily extended and customized to suit your need.













