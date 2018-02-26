# Making a Movie CLI

_You can see the latest version of the result of this tutorial on Github at [https://github.com/infinitered/tutorial-movie](https://github.com/infinitered/tutorial-movie)!_

In this tutorial, we'll make a Gluegun-powered command-line interface called `movie`. Before doing this tutorial, make sure you've followed the installation instructions in [Getting Started](/getting-started).

## Generate a new CLI

At your terminal prompt, run the following commands.

```
$ gluegun new movie --typescript

Generated movie CLI.

Next:
  $ cd movie
  $ npm link
  $ movie

$ cd movie
$ npm link
$ movie --help

movie version 0.0.1

  version (v)    -
  help (h)       -
  generate (g)   -
  movie          -
```

At this point, open the folder in your editor. You should see something like this:

![image of editor with movie CLI source](https://user-images.githubusercontent.com/1479215/36636025-dd2d7c94-1972-11e8-9a5b-344a97f6ddf3.png)

## Connect to IMDB's API

We want to hook into IMDB's API to find information about movies and actors. Luckily, there's a nice little [NPM package](https://github.com/worr/node-imdb-api) for that!

```
$ npm install --save imdb-api
```

In order to use the API, you'll need an API key. We are not going to hard-code our API key into the CLI source. Instead, we'll ask the user for an API key and then store it locally.

_You can get a free API key (with a 1,000/day limit) [here from OMDB](http://www.omdbapi.com/apikey.aspx?__EVENTTARGET=freeAcct)._

## Create an extension

In your CLI's `src/extensions` folder, make a new file, called `imdb-extension.ts`. Here's the source:

```typescript
// in src/extensions/imdb-extension.ts
import * as imdb from 'imdb-api'

module.exports = toolbox => {
  // memoize the API key once we retrieve it
  let imdbKey: string | false = false

  // get a movie
  async function getMovie(movieName: string): Promise<imdb.Movie | null> {
    const result = await toolbox.prompt.ask({ type: 'input', name: 'key', message: 'API Key>' })

    if (result.key) {
      return imdb.get(movieName, { apiKey: result.key, timeout: 30000 })
    } else {
      return
    }
  }

  // attach our tools to the toolbox
  toolbox.imdb = { getApiKey, getMovie }
}
```

The source is commented, so read through it. We define a couple functions in the extension and then attach an object to the toolbox with the `getMovie` function. This will be available to any commands we make.

## Create a command

Now that we have the tools, let's build a command. We want to be able to search for a movie and display information about it, something like this:

```
$ movie search "The Road"
```

If they don't provide the movie title, we'll prompt them for it. We'll also prompt them for the API key.

Create a file `src/commands/search.ts` and put the following contents into it:

```typescript
const API_MESSAGE = `
Before using the movie CLI, you'll need an API key from OMDB.

Go here: http://www.omdbapi.com/apikey.aspx?__EVENTTARGET=freeAcct

Once you have your API key, enter it below.

API KEY>`

module.exports = {
  name: 'search',
  alias: ['s'],
  run: async toolbox => {
    // retrieve the tools from the toolbox that we will need
    const { parameters, print, prompt, imdb } = toolbox

    // check if there's a name provided on the command line first
    let name = parameters.first

    // if not, let's prompt the user for one and then assign that to `name`
    if (!name) {
      const result = await prompt.ask({ type: 'input', name: 'name', message: 'What movie?' })
      if (result && result.name) name = result.name
    }

    // if they didn't provide one, we error out
    if (!name) {
      print.error('No movie name specified!')
      return
    }

    // now retrieve the info from IMDB
    const movie = await imdb.getMovie(name)
    if (!movie) {
      print.error(`Couldn't find that movie, sorry!`)
      return
    }

    // success! We have movie info. Print it out on the screen
    print.debug(movie)
  },
}
```

The command is well commented, so read through those to see what we're doing. Notice that we're using the `imdb.getMovie` function that we created in the `imdb-extension.ts` extension prior to this.

Run it on the command line and search for a movie. You'll notice that the extension asks for an API key -- enter the one you got from OMDB there.

```
❯ movie search                                                                                                        ✱
? What movie? The Room
API KEY>  <ENTER YOUR API KEY HERE>
```

If you did it right, you should get something like the following output:

```js
vvv -----[ DEBUG ]----- vvv
Movie {
  title: 'The Room',
  _year_data: '2003',
  year: 2003,
  rated: 'R',
  released: 2004-03-03T08:00:00.000Z,
  runtime: '99 min',
  genres: 'Drama',
  director: 'Tommy Wiseau',
  writer: 'Tommy Wiseau',
  actors: 'Tommy Wiseau, Greg Sestero, Juliette Danielle, Philip Haldiman',
  plot: 'In San Francisco, we follow Johnny, a man who has a girlfriend, Lisa, and also his best friend, Mark. Lisa has been cheating on Johnny with Mark and Johnny doesn\'t know! Will Johnny ever find out? Will Mark still be Johnny\'s best friend?',
  languages: 'English',
  country: 'USA',
  awards: '1 win.',
  poster: 'https://images-na.ssl-images-amazon.com/images/M/MV5BMTg4MTU1MzgwOV5BMl5BanBnXkFtZTcwNjM1MTAwMQ@@._V1_SX300.jpg',
  ratings:
   [ { Source: 'Internet Movie Database', Value: '3.6/10' },
     { Source: 'Rotten Tomatoes', Value: '26%' },
     { Source: 'Metacritic', Value: '9/100' } ],
  metascore: '9',
  rating: '3.6',
  votes: '56,264',
  imdbid: 'tt0368226',
  type: 'movie',
  dvd: '17 Dec 2005',
  boxoffice: 'N/A',
  production: 'Chloe Productions',
  website: 'http://theroommovie.com/',
  response: 'True',
  series: false,
  imdburl: 'https://www.imdb.com/title/tt0368226' }
^^^ -----[ DEBUG ]----- ^^^
```

Success!

## Save the API key

We don't want to ask the user for an API key every time. So, if it's a valid API key, we want to save it to the user's filesystem and retrieve it on every load.

We also don't want to be asking for user input in our extension. That's the role of the command. (Read more about this in [Guide: Architecting Your Gluegun CLI](/guide-architecture).)

Let's go back to our IMDB extension and add a few tools. Here's the result:

```typescript
import * as imdb from 'imdb-api'

module.exports = toolbox => {
  const { filesystem } = toolbox

  // location of the movie config file
  const MOVIE_CONFIG = `${filesystem.homedir()}/.movie`

  // memoize the API key once we retrieve it
  let imdbKey: string | false = false

  // get the API key
  async function getApiKey(): Promise<string | false> {
    // if we've already retrieved it, return that
    if (imdbKey) return imdbKey

    // get it from the config file?
    imdbKey = await readApiKey()

    // return the key
    return imdbKey
  }

  // read an existing API key from the `MOVIE_CONFIG` file, defined above
  async function readApiKey(): Promise<string | false> {
    return filesystem.exists(MOVIE_CONFIG) && filesystem.readAsync(MOVIE_CONFIG)
  }

  // save a new API key to the `MOVIE_CONFIG` file
  async function saveApiKey(key): Promise<void> {
    return filesystem.writeAsync(MOVIE_CONFIG, key)
  }

  // reset the API key
  async function resetApiKey(): Promise<void> {
    await filesystem.removeAsync(MOVIE_CONFIG)
  }

  // get a movie
  async function getMovie(movieName: string): Promise<imdb.Movie | null> {
    const key = await getApiKey()
    if (key) {
      return imdb.get(movieName, { apiKey: key, timeout: 30000 })
    } else {
      return
    }
  }

  // attach our tools to the toolbox
  toolbox.imdb = { getApiKey, saveApiKey, getMovie, resetApiKey }
}
```

We've added a few functions as well as the `MOVIE_CONFIG` file. We're using the `filesystem` tool from the toolbox.

Also update your command to ask for the API key:

```typescript
//
const API_MESSAGE = `
Before using the movie CLI, you'll need an API key from OMDB.

Go here: http://www.omdbapi.com/apikey.aspx?__EVENTTARGET=freeAcct

Once you have your API key, enter it below.

API KEY>`

module.exports = {
  name: 'search',
  alias: ['s'],
  run: async toolbox => {
    // retrieve the tools from the toolbox that we will need
    const { parameters, print, prompt, imdb } = toolbox

    // check if there's a name provided on the command line first
    let name = parameters.first

    // if not, let's prompt the user for one and then assign that to `name`
    if (!name) {
      const result = await prompt.ask({ type: 'input', name: 'name', message: 'What movie?' })
      if (result && result.name) name = result.name
    }

    // if they didn't provide one, we error out
    if (!name) {
      print.error('No movie name specified!')
      return
    }

    // check if we have an IMDB API key
    if ((await imdb.getApiKey()) === false) {
      // didn't find an API key. let's ask the user for one
      const result = await prompt.ask({ type: 'input', name: 'key', message: API_MESSAGE })

      // if we received one, save it
      if (result && result.key) {
        imdb.saveApiKey(result.key)
      } else {
        // no API key, exit
        return
      }
    }

    // now retrieve the info from IMDB
    const movie = await imdb.getMovie(name)
    if (!movie) {
      print.error(`Couldn't find that movie, sorry!`)
      return
    }

    // success! We have movie info. Print it out on the screen
    print.debug(movie)
  },
}
```

Running it again will ask for our API key. Subsequent runs won't ask for it anymore, and you should see a `.movie` file in your home folder that contains your API key.

## Add another command

We want to be able to reset the API key if we need to. Let's create another command for `movie api reset`, located at `src/commands/api/reset.ts`. Add this code:

```typescript
module.exports = {
  name: 'reset',
  run: async toolbox => {
    // retrieve the tools from the toolbox that we will need
    const { prompt, imdb, print } = toolbox

    // confirmation, because this is destructive
    if (await prompt.confirm('Are you sure you want to reset the IMDB API key?')) {
      // delete the API key
      await imdb.resetApiKey()
      print.info('Successfully deleted IMDB API key.')
    }
  },
}
```

We also need to update the imdb extension:

```typescript
// src/extensions/imdb-extension.ts

  // omitted a lot of code ...

  // reset the API key
  async function resetApiKey(): Promise<void> {
    await filesystem.removeAsync(MOVIE_CONFIG)
  }

  // attach our tools to the toolbox
  toolbox.imdb = { getMovie, resetApiKey }
}
```

At the command line, run `movie api reset` and you should see the prompt to delete the key.

## Additional Exercises

As additional exercises, try doing these things with your new CLI:

1. Show a customized help screen with `movie help`. (Hint: add a `src/commands/help.ts` and remove the `.help()` in `src/cli.ts`)
2. Show nicer output from the `search.js` command. (Hint: replace `console.debug(movie)` with a table using `print.table()`)
3. Add the ability to search for actors

## Notes

* We [intend to improve](https://github.com/infinitered/gluegun/issues/361) the generated CLI with `gluegun new mycli --typescript` to include better TypeScript bundling for production. Look for that in the future, or, better yet, [help us do that](/contributing).
* The architecture of the above CLI works, but as it grows, you'll want to start organizing it a little better. Read [Guide: Architecting Your Gluegun CLI](/guide-architecture) to learn more.

_Questions? Jump in our [Infinite Red Community Slack](http://community.infinite.red) in the #gluegun channel and ask away!_