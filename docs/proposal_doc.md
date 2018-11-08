# Profile API Design Discussion

## Overview
This is just a quick proposal for the structure and interface of an API for the profile system. In a nutshell, I'm suggesting having an API server that provides profile data via a JSON REST API. It'll be backed by a database (probably MongoDB) and image storage (AWS S3). It will need to integrate with the auxillary apps we build, and will most likely need to use the same database instance as whatever search engine we decide to use.

I would appreciate any input you have. In particular, I urge you to **please** read the [Search Integration Concerns](#search-integration-concerns) and [Badge System Concerns](#badge-system-concerns) sections. These are big topics, and having more brains on the matter is going to be vital to getting an elegant solution quickly.

Thanks for your time - I look forward to hearing your thoughts.

_- Patrick_

## Topics

- [Profile API server responsibilities](#Profile-Service-Responsibilities)
- [Outstanding questions](#questions)
- [Proposed endpoints + methods](#api-endpoints)
- [Implementation + architecture](#implementation-architecture)
    - [API Server](#api-server)
    - [Normal Profile Data](#normal-profile-data)
    - [Profile Images](#profile-images)
    - [Search Integration Concerns](#search-integration-concerns)
    - [Badge System Concerns](#badge-system-concerns)

## <a href="#responsibilities">Profile Service Responsibilities</a>
I've narrowed down the following responsibilities, but obviously we may add more later if needed.
- Provide profile data for our users as a JSON REST API.
- Provide user data in a format that the search service can ingest.
- Deliver "badge data" from 3rd party apps.

## <a href="#questions">Outstanding Questions</a>
these are the questiosn I'm milling over right now, which I'd love input on.
- What's the best way to integrate search functionality? (see [discussion](#search-integration-concerns))
- What's the best way to integrate badge functionality? (see [discussion](#badge-system-concerns))
- How are we managing permissions for external apps? Do we actually need to? (same discussion as above)

## <a href="#api-endpoints">API Endpoints</a>
This is a quick sketch of the basic endpoints this API would probably provide. If you see anything that's missing, please let me know!
- [.../user/{user-id-or-name}](#user/)      GET, POST
- [.../user/{user-id-or-name}/name](#user/name/) GET, POST
- [.../user/{user-id-or-name}/bio](#user/bio/) GET, POST
- [.../user/{user-id-or-name}/img-url](#user/img-url/) GET
- [.../user/{user-id-or-name}/image](#user/image/) POST
- [.../user/{user-id-or-name}/badges](#user/badges/) GET

## <a href="#implementation-architecture">Implementation + Architecture</a>
There are a few core design issues I've found so far. I have discussions of each of them further down.
- Providing the API
- Storing and providing normal profile data (name, bio, etc.)
- Storing and handling images
- Search integration
- Dealing with badges + other app integrations

For infrastructure components, we're thinking in terms of the following main components:
- An API server (NodeJS, hosted on Heroku)
- Database (MongoDB, run through one of Heroku's partners)
- Image storage (S3 bucket)

NodeJS because we're pretty familiar with it, likewise for S3. MongoDB is still open, but it should be fairly simple to use.

### <a href="#api-server">API Server</a>
The API server's job is to expose the API to the web, and to orchestrate requests by communicating with the database, AWS S3, and other connected apps. We are currently assuming we'll write it in NodeJS, using express.

### <a href="#normal-profile-data">Normal Profile Data</a>
Normal profile data (name, biography, etc.) will be stored in the server's dedicated database (we're currently thinking MongoDB). Please note that we're going to have to treat images and text differently just due to storage concerns (I don't think we want to be storing images in the database). I've got a quick discussion + proposed solution on this in the next section.

Here's a breakdown of the profile data elements I think we'll probably need.

```
{
    lastUpdated:    123456789874,     // Server timestamp
    displayName:    'john doe',
    biography:      'lorem ipsum blah blah blah',
    imgUrl:         'https://some-s3-bucket/some-img.jpg',
    badges: [
        {
            iconUrl:    'https://wherever.you.put/your-icon.jpg',
            text:       'JavaScript Guru (88/100)',
            href:       'https://other.app.endpoint/'
        },
        ...
    ]
}
```

### <a href="#profile-images">Profile Images</a>
Obviously, we probably don't want to be storing images in our MongoDB, or it's likely to fill up really fast. Cam and I discussed it, and we figured we'll probably just store images in AWS S3. This lightens the load in the database, but means a few things:

- We need to facilitate uploading images through the node server to S3 (use AWS SDK on the node server, plus store credentials and set up IAM role).
- We need to ensure each of those images is unique.
- We need to store links to those images (at the very least).
- We need to decide how people are going to retrieve those images (separate download from the S3, or routed through the node server).

My inclination is to allow users to upload images by sending **POST** requests to `.../user/{user-id}/image`, but only allow **GET** requests to `.../user/{user-id}/img-url` and forcing them to go download the image separately. I'm open to suggestions though!

Regarding the storage and naming conventions for the images, we obviously need to make sure each user image has a unique name. This could be whatever primary key we use for the user. However, if that primary key is something like e-mail or username, we should probably anonymize it as a number or something, and just store that number in our DB. Also, a quick point on this is that a system like this would limit each user to one profile picture (I'm 100% okay with this, but please let me know if you think that might be a problem).

### <a href="#search-integration-concerns">Search Integration Concerns</a>
While at this time, we don't know how the search system is actually going to work (what libraries we'll use, etc.), we *do* know that at the very least, it's going to need fast access to basically all of our user data, not only from the profile system, but also from the auxillary apps.

I don't have any sort of easy answer for this yet, but I have a couple of ideas, and I'd love to hear your thoughts and suggestions. Warning, the following is a bit lengthy.

#### Profile as Aggregator
My initial thought is that, as we're going to need all the data in one place anyway, we could use the Profile Server + API as an aggregator. If we don't do it on the profile server, we'll still need to pull all that data to somewhere.

#### Searching Data in Multiple Places

Assumptions:

1. The requirements state that _"your app search functionality must search users' descriptions by applying filtering on badges."_
2. We're required to use at least two back-end stacks, which I think means we can't keep all our data in one place.
3. It's a hunch, but I'm inclined to say that searching is a really time-complex thing to do.

I don't think we can sensibly make hundreds of API calls to each separate app every time someone runs a search. My initial gut reaction is that we're probably going to need to cache data from each of the separate apps in one place, and that the caching will need happen somewhere close to wherever the search engine is running.

A couple of ways we could do this:

- Cache in the profile server's database. Store a "last updated" field for every user, and update it any time the user changes something on their profile, or new badge data is received (still querying each auxillary badge app every time a user's profile is requested).
- Same as above, but have each auxillary badge app notify the profile server when they update. This would probably mean fewer network API calls (as the profile server never queries those other apps), but is obviously a lot more complicated.

In both of these cases, we're storing data in the profile server's DB. This means that we'll would also probably need to have the search system run on the same DB. That's not necessarily an issue, but just something to think about, as it'll mean the Profile API team will need even closer communication with the Search Engine team.

Another option:

- Search server maintains its own database, and caches everything there. It can maintain a record of when it last got new information from the Profile Server and request all updates that occured since it last checked. This would make the search engine a little out of date, but I don't think that matters. I'm more concerned about the complexity of it, as we need to store last-update data in a couple of places, and possibly transfer huge amounts of data between the two servers (seems a bit redundant).

**Recommendation**, based on current knowledge:

I would recommend going with option 1, and just making sure that we don't dig into building the profile database until we have a solid report from the search team on what technologies they want to use.

### <a href="#badge-system-concerns">Badge System Concerns</a>
As you probably know, the concept behind the badge system is to allow other apps within our ecosystem to display relevant data on user profiles. This is basically just going to be a short snippet comprised of three things:

1. An icon.
2. Some text ("JavaScript Champion" or whatever).
3. A link back to the service that's displaying the data.

Now, the requirements state that:

> _Each app can request access to the following:_
>- _Badge frame_
>- _User landing page_
>- _users information_

The words "request access to," along with what I remember from Amir's lecture, lead me to think that this probably means some sort of permissions system (as in, each app requests a user's permission to access another part of the app).

With that said, here are the assumptions I've made so far, along with what I think are probably the biggest issues:

**Assumptions:**

- The original badge data needs to come from a auxillary API.
- The profile server needs, at a minimum, a list of the source services, and endpoints to hit when querying them.

**Questions:**

- Should the badge data should be retrievable through the profile API? (i.e., when you request `.../user/{user-id}/badges`, do you get the actual badge data, or a list of endpoints to query?)
- Where should the formatted badge data be stored? (On profile db? On auxillary app's db? Both?)
- Does the profile server query the auxillary services, or do they push to the profile API? The former requires each extra API we build to have a similar interface.
- How do we store the list of auxillary services attached? On a global basis? On a per-user basis?
- How do we authenticate between APIs? It's no good to allow any old POST request to push data to a user's profile.
- If we have a permission system, where is that handled?

