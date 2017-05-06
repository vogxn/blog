+++
date = "2017-05-06T10:00:58+05:30"
title = "Pitfalls in session management"
tags = ["golang", "gorilla", "sessions", "webapps"]

+++

This is a quick post that talks about the usage of sessions in web
applications. Sessions is not a new topic for those who've used the basic web
applications that require a login. Sessions are necessary to overcome the
stateless nature of HTTP. They do this by utilising the cookie mechanism of
most modern web browsers.

### A Sessions Analogy

In simple terms, a session is used to identify and resume conversation with the
service where one left off previously. Anecdotally, imagine you went to a bank
to fill out a withdrawal form. You fill the form halfway only to be told by the
teller that you've forgotten to include a copy of your proof of identity. So
you go back and return later to complete the withdrawal. Now, there are a few
ways to do this:

1. You take your form with you and return with the proof of identity
1. You let the teller keep the incomplete form and return with the proof of identity
1. The teller keeps your incomplete form and gives you a token so he can retrieve the form from his filer later 


Granted, this is a little contrived. But in cases 1. and 2. there is potential
for data loss (_you or the teller loses the form_). And since we all hate
filling forms, the token approach is better suited as the form is kept in
(_presumably_) safekeeping.

### Session storage

In scenario 1. the client carrying the form data back is problematic because

1. There is too much data to carry. Browsers restrict 4kB of data in cookies.
2. The form data could be lost. *\*Groans\** - fill your form again
3. The data could be manipulated by a malicous third party

So this option of storage is not viable and insecure for the most part. 

In scenario 2. if the teller was a different individual, there is going to be a
lot of scurrying about trying to locate your form. In a web application there
is no guarantee that the request is served by the same node each time. We can
simply discount this method as infeasible.

In scenario 3. the token is a simple key based mechanism to perform a look up
in the filer storage. You could go to a different teller and they would still
be able to locate your form. And you don't have to carry a ton of data back
with you. Even within a client-server situation this is the favoured approach.


### Implementations

I was going through the [gorilla
sessions](https://gorillatoolkit.org/pkg/sessions) pkg and was left
[confused](https://github.com/gorilla/sessions/issues/114) by the default
`FilesystemStore` and memory based `CookieStore`. Both of these are cookie
driven implementations of session storage. A look at the interface and examples
on the gorilla website would lead you into saving your form data (encoded
though) as part of the cookie sent back to the client. Here's the session save
setting the cookie in the response.

```Go
// Save adds a single session to the response.
func (s *CookieStore) Save(r *http.Request, w http.ResponseWriter,
    session *Session) error {
    // Saves session and values
    encoded, err := securecookie.EncodeMulti(session.Name(), session.Values,
        s.Codecs...)
    if err != nil {
        return err
    }
    http.SetCookie(w, NewCookie(session.Name(), encoded, session.Options))
    return nil
}
```

As you can see the `session.Values` corresponds to the form data saved _if_ you
follow the examples. This leads to
[abuse](http://wonko.com/post/why-you-probably-shouldnt-use-cookies-to-store-session-data)
of session management in frameworks in any language. 

However, the redis and couchdb implementations of the gorilla sessions do not
have this issue. The redis implementation for instance only saves the
`session.ID` in the cookie for later lookup in redis

```Go
// Save adds a single session to the response.
func (s *RediStore) Save(r *http.Request, w http.ResponseWriter, session *sessions.Session) error {
    // Marked for deletion.
    if session.Options.MaxAge < 0 {
        if err := s.delete(session); err != nil {
            return err
        }
        http.SetCookie(w, sessions.NewCookie(session.Name(), "", session.Options))
    } else {
        // Build an alphanumeric key for the redis store.
        if session.ID == "" {
            session.ID = strings.TrimRight(base32.StdEncoding.EncodeToString(securecookie.GenerateRandomKey(32)), "=")
        }
        if err := s.save(session); err != nil {
            return err
        }
        // Only saves session.ID
        encoded, err := securecookie.EncodeMulti(session.Name(), session.ID, s.Codecs...)
        if err != nil {
            return err
        }
        http.SetCookie(w, sessions.NewCookie(session.Name(), encoded, session.Options))
    }
    return nil
}
```

I'm not entirely sure of the intent of the Gorilla authors why the default
implementations differ this way though. I must reach out to them.

### Security

One topic I didn't touch upon is that of security. You will notice that most
sessions are encrypted and are validated before use. In the teller scenario, if
we flip the roles there is no guarantee that the data has not been tampered
with. This is why the token and the data need to be secured in all cases.
Luckily for us the Gorilla toolkit provides the `securecookie` implementation
that does this with only a few lines of code.

