# Application Notes

**WARNING**
-> Some stuff here has changed, the correct reference is the real notes.md on the backend repository (which is not public btw).

## Database

### Token authentication system
Tokens have to be limited in time, bound to a user. We could also bind to an IP address but let's start without that for the moment.

* token (primary key) - varchar
* user_id
* expires - datetime or equivalent
* original_ip_address - varchar
* last_ip_address - varchar

-> We need an archive table for this.

TODO:
There should be some kind of counter for failed auth attempts to possibly blacklist IPs.

To simulate a call with authentication we should be able to do something like this:
```
curl --header "Authorization: bearer LSMFKJDSkfjdskfh3473dh22" http://localhost/v1/cal/events
```

## API structure
Main URLs:
* /v1/cal/ -> Mount calendar stuff here.
* /v1/address-book/ -> Mount address book stuff here.
* /v1/account/ -> Authorizations, authentications, etc.

We're going to have one controller per element here, and mount them on the right URL.

I'm not super sure how to provide the POST parameters. I might do it in JSON.

### Account calls
This is how we authenticate, for a start.

All these calls have to be made in HTTPS or it's kind of easy to steal data here.

We could add calls to enable users to set personal information, preferences, etc.

#### /token - POST
Requires an authentication token. Username and password to give in the request. This could or could not invalidate older tokens for the same user.
If we want the app to be possibly accessed at different places by the same account we need to keep the other unexpired tokens.

Testing it:
```
curl http://localhost/v1/account/token -d '{"login":"admin","password":"test"}' -H 'Content-Type: application/json'
```

#### /token - GET
Check validity of token given in the Authenticate: bearer header. Or something like that.

Return information on token + information on the logged in user.

This call should have strict rate limits as it's a security issue.

#### /token - DELETE
Remove token. Can be bound to a logout system.

Testing:
```
curl -X "DELETE" http://localhost/v1/account/token/<INSERT_TOKEN>
```

### Calendar calls
We need a call with user defined time range. Which should be limited in some way.

We could have a call to check reminder status at some point.

#### /events - GET
Authenticated. Pretty much like every call so I'm going to stop mentioning it.

We need some sort of start and end parameters. There should be a default time range if nothing is given.

The parameters are given as query parameters like ?start=START&end=END

To get the events for just a day, it's important for end to be the next day at 00:00. That's because the comparison in the backend is made on the dates and not time of day or whatever.

Making a full test call:
```
curl --header "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad" "http://localhost/v1/cal/events?start=2017-08-12&end=2017-08-22"
```

#### /events/{id} - GET
Gets a specific event in JSON format. I'm not sure if the frontend will ever need this.

Using an ID that doesn't exist will yield a 403 and not a 404. The logic is that an event that doesn't exist also doesn't belong to you in any cases.

```
curl --header "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad" "http://localhost/v1/cal/events/38473"
```

#### /events - POST
This will be how we create events.

We need an attribute to know if modifying or not. Let's use the event ID being present or not for that.

Returns an HTTP error if failed ;
Returns the event in JSON format if succeeded.

When editing all the fields are mandatory, any absent field will null or void that field in database or produce an error.

Leaving personId out completely:
```
curl http://localhost/v1/cal/events -d '{"personName":"Dude Mc Dude","location":"somewhere","reminderSms":true,"reminderEmail":false,"date":"2017-08-06","start":"10:00","end":"11:30"}' -H "Content-Type: application/json" -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

* TODO Test with a valid person ID, an invalid person ID, and a person ID that belongs to another user.
* TODO Test with missing fields.
* TODO Test with invalid dates.

Testing with a personId, inverting start and end (the backend should re-invert them):
```
curl http://localhost/v1/cal/events -d '{"personId":20,"personName":"Dude Mc Dude","location":"somewhere","reminderSms":true,"reminderEmail":false,"date":"2017-08-06","start":"17:00","end":"11:30"}' -H "Content-Type: application/json" -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```
-> This call will completely ignore "personName".

Posting an event with a date in the past is accepted but is never supposed to create any reminder.

#### /events - DELETE
Obvious. The evend id is given in the URL. Example: /events/43

Returns a JSON with the 'result' key, value 'OK' if it worked.
HTTP error otherwise.

```
curl -X "DELETE" http://localhost/v1/cal/events/11 -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

### Address book calls

#### /persons - GET
Authenticated call, get the list of persons for this user. The token tells us which user it is. This is the case for all the calls on "persons".

GET parameters:
* order: default is asc, can be set to desc

We could limit this call using a starting letter to the first name or last name of the person -> This is going to be another call.

The problem is that pagination will require using LIMIT. I have to investigate PDO and LIMIT. I could create this call later and first work on the alphabetical stuff.

We should always sort by lastname only.

There is already a GET class /persons/{number}, which is the single person detail call.

What we could do is:
* /persons -> main call get a limited list + the amount of pages, and total count of persons
* /persons/page/{number} -> makes a LIMIT x, y query ; PAGES VALUES START AT 1
Those are two different callbacks when it comes to Silex. But it's more beautiful than a query parameter.

These two calls return results as such:
```
{"pages": 3, "currentPage": 1, "resultCount": 30, "totalCount": 90
  "persons": [
    {person1...},
    {person2...},
    {person3...}
  ]
}
```

If the array is empty:
```
{"pages": 0, "currentPage": 1, "resultCount": 0, "totalCount": 0, "persons": []}
```

Testing base call:
```
curl http://localhost/v1/address-book/persons -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

Testing page call:
```
curl http://localhost/v1/address-book/persons/page/2 -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

* /persons/starting-with/{string}
-> We strip a limited amount of characters from this string. I say we start with 2.

Return something like:
```
{"resultCount": 30, "persons": [
  {person1...},
  {person2...},
  {person3...}
]}
```
This call doesn't have any pagination.
```
curl http://localhost/v1/address-book/persons/starting-with/ka -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

I haven't tested the amount of letters limitation nor the special characters like having some kind of URL escaped "%".

#### /persons/id
Get the info for this person in JSON.

We need to add a boolean field telling the app if this person has appointments linked to them.

I already have a function in the manager to check for that.

Let's say it's going to be:
'hasAppointments'

Testing:
```
curl http://localhost/v1/address-book/persons/30 -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

The frontend is supposed to use hasAppointments to display a warning if user is trying to delete that person.

#### /persons - POST
Create a new person for the authenticated user. We know if we're trying to modify an existing item by the presence of an id.

This is very similar to the POST method for events.

Returns the person object as JSON if it worked or HTTP error 500 if insertion didn't work.

When editing all the fields are mandatory, any absent field will null or void that field in database or produce an error.

Fields:
* personId (determines if updating existing entry)
* lastname 
* firstname 
* birthdate 
* phoneMobile 
* phoneHome
* phoneWork 
* email
* languageId

Creating a person example:
```
curl http://localhost/v1/address-book/persons -d '{"lastname":"Simon","firstname":"Franck","birthdate":"1984-10-12","phoneMobile":"0477333333","email":"test@dkvz.eu","languageId":1}' -H "Content-Type: application/json" -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

Updating a person:
```
curl http://localhost/v1/address-book/persons -d '{"personId":35,"lastname":"Pulleur","firstname":"Pull","birthdate":"1990-10-12","phoneMobile":"04773322333","email":"testage@dkvz.eu","languageId":1}' -H "Content-Type: application/json" -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

#### /persons - DELETE
You guessed it. This call requires consistency checks because appointments could be linked to this person.

At the moment it's setting personId to null on any appointment that had it.

```
curl -X "DELETE" http://localhost/v1/address-book/persons/20 -H "Authorization: bearer 61b8b405881e69ec9401ba485567300caac881ad"
```

## TODO
* Logging failed authentification attempts with possible temporary ban of IP addresses.
* Rate limiting?
* There is a style error in the database, reminderSms and reminderEmail should be named reminder_sms and reminder_email in there.
* I'm using (int) converters in my code, what's the upper bound for int in PHP? Might be reached by an enormous amount of appointments.
* Create test scenario for each API, like a full create / view / edit / (view) / delete cycle.
* Check how error handling works in prod, I might have to define an error handler.
* In table person, lastname has to be mandatory and not null.
* Try to SQL-ingest my app. Try it while leaving a vulnerability on purpose on day to test the common pentest tools.
* When we get individual events, the manager class is adding user_id and duplicating displayName as title, the controller should be doing that instead of the manager class.
* It would make easier for Javascript stuff to have all the API endpoints respond with "result":"OK","object":{}} or something.
* There should be an index on lastname for table person, we're using this for LIKE queries.