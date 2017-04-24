+++
date = "2017-04-19T17:32:37+05:30"
title = "Tweetpost"
tags = ['tweetpost', 'golang', 'webapp']

+++

I was first exposed to the Go language back in 2013 when I was working with [Apache
Cloudstack](https://cloudstack.apache.org/). At the time, I was using [Mitchell
Hashimoto's](https://github.com/mitchellh)
[Packer](https://github.com/hashicorp/packer) for spinning up images in AWS.
However, a switch of jobs took me in a different direction and I never delved
deeper. Now, I've finally found some time to experiment with Go again.

### Language Review

I went back to review the language using the excellent
[golang](https://golang.org) website. The golang
[tour](https://tour.golang.org/) was my first stop where I found the examples
have been improved - channels and concurrency are explained much better. But
this isn't nearly enough reading to build anything useful yet. As recommended,
I went through the wonderful applied language tutorial at
[appliedgo.net](https://appliedgo.net/) and the standard libraries, all the
while being reminded that [Idiomatic
Go](https://golang.org/doc/effective_go.html) is the style that all Golang
programmers should aspire to write.

### The Project

I wanted to apply my learning to build a web application. I've traditionally
built more command line driven tools over the years and have lost touch with
the web.

My pet project was very simple, in fact, it wasn't even something novel. But
what the heck? I wanted to build it with Go. And hence we have [Tweetpost](https://github.com/vogxn/tweetpost.git).
Tweetpost is yet another of those online tools to help you rant on the internet
via your Twitter account. In that sense it's probably a poor version of
[Tweetstorm](http://tweetstorm.io).


## Part 1: Tokenizer

The first part was simply breaking down text into tokens so the total length
did not exceed Twitter character limits. Now you can write this using basic
arrays and string manipulation without much fuss. But Go's interfaces are
powerful. The Go community will often stress on the virtues of the
[`Reader`](https://golang.org/pkg/io/#Reader) and
[`Writer`](https://golang.org/pkg/io/#Reader) interfaces and how widely they
are applied through the standard library for a myriad functions. So I went
looking to see if there was something that could assist in text tokenization.
And there it was - [`Scanner`](https://golang.org/pkg/text/scanner/)

The `Scanner` interface is implemented by `bufio` package to break text into
tokens of words, lines and runes. In order to adapt this for splitting tweets I
had to implement a [`SplitFunc`](https://golang.org/pkg/bufio/#SplitFunc)
function. The signature of which looks as shown:

```Go
// ScanTweets is a split function for a Scanner that returns a sequence of
// tweets, each tweet not more than TWEET_CHAR_LIMIT(140) characters in length.
func ScanTweets(data []byte, atEOF bool) (advance int, token []byte, err error) {
   ...
}
```

The implementation is a slight modification of the `ScanWords` function. We
detect word boundaries but keep appending the word to a tweet slice that we are
building. The token boundary is imposed by checking that the append does not
cause us to exceed the twitter character limit. The next token scan begins when
we return the tweet built thus far.

Now we can start breaking any text into tweets with a simple `scanner.Scan()`
iteration as shown below.

```Go
var longTweet = `This is a long tweet that should be split into two separate
tweets using the splitter function of this project.  This should validate the
basic splitting check and perform a simple functional test.`

var scanner = bufio.NewScanner(strings.NewReader(longTweet))
scanner.Split(ScanTweets)
for scanner.Scan() {
  fmt.Println(scanner.Text())
}
```

As the tests show, the scanner takes anything that resembles a `Reader`
interface. So we can convert text files too into tweets using this newly
written function.

## Part 2: Web application

I started this blog using [Hugo](https://gohugo.io/) - a static website
generator written in Go. When doing that I encountered
[templates](https://golang.org/pkg/html/template/). Templates as the name
suggests is a simple templating engine that is part of the Go standard
libraries. The first implementation of Tweetpost therefore used the
`html/template` utilities to render the various pages with a simple form
`POST`. However, most modern web applications are written to do client side
rendering to optimize bytes transferred. 

So I decided to try and learn the full-blown MVC way to write my toy application.
Information about Go specific layouts for an MVC app was littered with a lot of
opinions and frameworks (which the Go community seems to abhor). The
[awesome-go](https://github.com/avelino/awesome-go#web-frameworks) listings
seem to have at least two dozen different frameworks
and toolkits for the purpose. I picked something simple however.

### Innards

- **Layout**: Used the examples by [Jon Calhoun](https://www.calhoun.io/creating-controllers-views-in-go/)
- **Routing**: For routing and the REST API I used [httprouter](https://github.com/julienschmidt/httprouter) for its simplicity
- **Response/Templates**: `net/http` for response writing with minimum use of `html/template`
- **Client Side**: [Bootstrap](https://getbootstrap.com) and jQuery
- **Configuration**: Configuration is read and managed by [Viper](https://github.com/spf13/viper)
- **OAuth**: Twitter OAuth is _attempted_ using [`mrjones/oauth`](https://github.com/mrjones/oauth)

I've structured my MVC layout as shown. There may be many ways to do this but
the split I've used is such that each package is self-contained and performs
all the functions in a single area. For eg: all the authorization with OAuth is
done in the `auth` package. Page views are handled in the `views` package and
routing in the `controllers` package

```
server
├── auth
│   └── credentials.go
├── config
│   └── settings.yaml
├── controllers
│   ├── post.go
│   └── static.go
├── main.go
├── static
│   ├── css
│   ├── favicons
│   └── js
├── templates
│   └── layouts
│       ├── content.tmpl
│       ├── footer.tmpl
│       ├── header.tmpl
└── views
    └── page.go
```

You can start the web app by running `server/main.go`. All the configuration is
read from the `config` directory where I store the OAuth keys. The
`auth` package encapsulates the `TwitterAuth` type which is a reductive
type from the full blown set of details needed for OAuth. After performing the
OAuth dance an `HttpClient` is created with which you can access the Twitter API.

The two routes that are exposed by the REST API are `/split` and `/tweet`.
`split` will break down the text into tweets using the `ScanTweets(..)`
function while `tweet` is expected to perform OAuth and post the tweets.
Currently twitter OAuth is broken so this is not supported. But the necessary
OAuth workflow is present in the `auth/credentials.go` file.

_Note: The app is incomplete because Twitter's 3-legged authorization is broken._ [^twitterbrokenoauth]

[^twitterbrokenoauth]: Twitter authorization fails [CORS preflight](http://stackoverflow.com/q/35879943/1297846) in a browser


Following shows a simple cURL call to break down tweet text.

```Bash
➜  ~ curl -sL -X POST --data '{ "text": "Lorem ipsum dolor sit amet,
consectetur adipiscing elit. Phasellus sed cursus turpis. Integer at efficitur
diam. Suspendisse accumsan felis in mollis bibendum. Phasellus aliquam
tincidunt mauris, vitae porttitor tellus laoreet et. Suspendisse eu purus
dolor. Vivamus eu scelerisque tellus, in imperdiet justo. Aliquam erat
volutpat. Cras sit amet porta mi, at pharetra felis. Morbi congue, ex in
blandit iaculis, eros eros suscipit nibh, vitae vehicula nulla odio nec justo.
Integer augue tortor, tempor eget purus non, fringilla dapibus augue. Curabitur
tristique dui eget mattis faucibus. Nulla sed dictum risus, vitae hendrerit
eros."}'  http://localhost:8080/post | jq .
[
  {
    "text": "1/ Lorem ipsum dolor sit amet, consectetur adipiscing elit. Phasellus sed cursus turpis. Integer at efficitur diam. Suspendisse accumsan"
  },
  {
    "text": "2/ felis in mollis bibendum. Phasellus aliquam tincidunt mauris, vitae porttitor tellus laoreet et. Suspendisse eu purus dolor. Vivamus eu"
  },
  {
    "text": "3/ scelerisque tellus, in imperdiet justo. Aliquam erat volutpat. Cras sit amet porta mi, at pharetra felis. Morbi congue, ex in blandit"
  },
  {
    "text": "4/ iaculis, eros eros suscipit nibh, vitae vehicula nulla odio nec justo. Integer augue tortor, tempor eget purus non, fringilla dapibus"
  },
  {
    "text": "5/ augue. Curabitur tristique dui eget mattis faucibus. Nulla sed dictum risus, vitae hendrerit eros."
  }
]
```

Here's a screenshot of the application.
{{< figure src="/img/tweetpost.png" >}}

Checkout the project on github -
[Tweetpost](https://github.com/vogxn/tweetpost.git). Since this is my first Go
webapp, I welcome comments and feedback on whether I've stuck to Idiomatic Go
syntax and convention.


