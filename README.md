# Flask SCIM Server (Configured for use with Okta)
**NOTE:** This is not meant to be a fully-fledged, IDP-agnostic SCIM server. This server was designed specifically to integrate with Okta.

## Dependencies:
- [virtualenv](https://docs.python.org/3/library/venv.html)
- [PostgreSQL](https://www.postgresql.org/)
- [ngrok](https://ngrok.com/)

## Setting up the Flask app and Postgres DB
1.  Clone the repo, open terminal and ```cd``` into the project root.
2.  Create a new virtualenv in the root folder with ```virtualenv env```
3.  Run the virtual environment with ```source env/bin/activate```
4.  Install necessary Python packages with ```pip install -r requirements.txt```
5.  Create a new Postgres database ```scim``` at ```postgresql://localhost/scim```. Enter the psql shell by opening a new terminal tab and typing ```psql postgres```. Create the DB with ```CREATE DATABASE scim;``` (Run ```\l``` to double check the database has been created)
6.  Go back to the terminal tab that is in the flask app root. Run the following commands to create migrations and tables in the ```scim``` database:
    - ```python manage.py db init```
    - ```python manage.py db migrate```
    - ```python manage.py db upgrade```
    
> Feel free to hop back to your postgres tab and run ```\c scim``` to navigate into the scim db, then ```\dt``` to see your new tables: ```groups```, ```users```, ```link```. (Link is a table facilitating the many-to-many relationship between users and groups)

7. Everything should be setup now to run the server locally. Finally run ```python app.py``` to do so. You should now have your SCIM server running on http://localhost:5000.

## Setting up ngrok to route requests from Okta to localhost
- Once you have ngrok installed, run ```./ngrok http 5000``` to create a tunnel from ngrok to your http://localhost:5000. Copy the ```https``` Forwarding URL created by ngrok as you will need it later.

## Creating and configuring your Okta Application
- Now it's time to create a new SCIM integration in Okta. If your SCIM app(s) are already setup on the Okta side, feel free to skip ahead to **Test the SCIM Server**. There are two options that will work with this server, and I will ALWAYS recommend the first, which is using an Okta SCIM template application.

### Option 1: SCIM Template App

1. In your Okta dashboard, go to **Applications** -> **Applications**, then click the **Browse App Catalogue** button. Search for **SCIM 2.0 Test App (Header Auth)** and click the **Add** button once you have it pulled up.
2. In the **General Settings** tab, click **Next**.
3. We will set this up as a SWA application, so in the **Sign-On Options** tab, click **Secure Web Authentication**.
4. Click **Done**.
5. Tab over to **Provisioning** and click **Configure API Integration.**
6. Check **Enable API integration**.
7. In the Base URL field, paste in the ngrok url you generated above with **/scim/v2** appended to the end. In the API Token field, type **Bearer 123456789**. (Later on we will go over how to customize this auth header, but out-of-the-box, the SCIM server expects this value)
8. Click Test API Credentials and you should get a success message like the below:

![SCIM_1](https://i.imgur.com/iFaxU9G.png)

> You can navigate to `http://localhost:4040` to see the request from Okta on this request, as well as the response from the SCIM server.

![SCIM_2](https://i.imgur.com/gCht05S.png)

9. Click **Save**.

Now your **Provisoning** tab will look a bit different.

10. Click **Edit** next to **Provisioning to App** and check off:
    - Create Users
    - Update User Attributes
    - Deactivate Users
    
And **Save**.

![SCIM_3](https://i.imgur.com/KRZCbiw.png)

### Option 2: Enable SCIM Provisioning for Existing AIW App

> Feel free to skip over this section to **Test the SCIM Server** if you set your SCIM integration up above.

1. In your Okta dashboard, go to **Applications** -> **Applications**, then click the **Create App Integration** button. For this setup we will select **SWA - Secure Web Authentication**. Click **Next**.
2. You can put whatever you'd like for the **App Name** and **App Login Page URL**, as we will just be loking at the SCIM functionality and not the SWA aspect of this app. Click **Finish**.
3. In the **General** tab of the app, click **Edit** and toggle Provisioning from **None** to **SCIM**. Click **Save**.
4. Your app should now have a **Provisioning** tab. Tab over to it and fill out the integration settings like the below image. Make the Authorization header **123456789**. You can change this later in the SCIM flask app.

![SCIM_4](https://i.imgur.com/yaEl9FD.png)

5. Click **Test Connector Configuration** and you should see the following success confirmation:

![SCIM_5](https://i.imgur.com/OetQUDR.png)

6. At which point, you can now click **Save**.

> You can navigate to `http://localhost:4040` to see the request from Okta on this request, as well as the response from the SCIM server.

![SCIM_6](https://i.imgur.com/gCht05S.png)


7. Now your **Provisoning** tab will look a bit different. Click **Edit** next to **Provisioning to App** and check off:
    - Create Users
    - Update User Attributes
    - Deactivate Users
    
And **Save**.

![SCIM_7](https://i.imgur.com/QzzUgHk.png)

You should now be set on the Okta side to start testing the SCIM server.

## Test the SCIM Server
>**Note**: I am using the SCIM template integration in the below steps. If you are using the AIW version, there may be subtle differences in some of the calls. More info [here](https://developer.okta.com/docs/reference/scim/scim-20/).

- Navigate to http://localhost:4040 to see all the requests and responses taking place between Okta and the SCIM server. I will be truncating mine a bit in the below examples for brevity.

- From the section above, you saw that we setup our SCIM integration with **Bearer 123456789** in the authorization header. You can change this header to whatever you'd like in the **app.py** file at the following line of function **auth_required**:

`if request.headers['Authorization'].split('Bearer ')[1] == '123456789':`

Replace '123456789' with whatever unique value you'd like and make sure to update this in the provisioning tab of your Okta integrations.

### Assign a User
Under the **Assignments** tab, click **Assign** -> **Assign to People**. I assigned user **obi-wan.kenobi@iamciam.dev**. Here is a look at the requests and responses from Okta to my SCIM server:

 - *Okta Request:*
```
GET /scim/v2/Users?filter=userName%20eq%20%22obi-wan.kenobi%40iamciam.dev%22&startIndex=1&count=100
```
 - *SCIM Server Response:*
```
200 OK
```
```
{
    "Resources": [],
    "itemsPerPage": 0,
    "schemas": [
        "urn:ietf:params:scim:api:messages:2.0:ListResponse"
    ],
    "startIndex": 1,
    "totalResults": 0
}
```
 - *Okta Request:*
```
POST /scim/v2/Users
```
```
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "obi-wan.kenobi@iamciam.dev",
    "name": {
        "givenName": "Obi-Wan",
        "familyName": "Kenobi"
    },
    "emails": [
        {
            "primary": true,
            "value": "obi-wan.kenobi@iamciam.dev",
            "type": "work"
        }
    ],
    "displayName": "Obi-Wan Kenobi",
    "locale": "en-US",
    "externalId": "00use6vjvehodmsQb4x6",
    "groups": [],
    "password": "gl^m&qWZ",
    "active": true
}
```
 - *SCIM Server Response:*
```
201 CREATED
```
```
{
    "active": true,
    "displayName": "Obi-Wan Kenobi",
    "emails": [
        {
            "primary": true,
            "type": "work",
            "value": "obi-wan.kenobi@iamciam.dev"
        }
    ],
    "externalId": "00use6vjvehodmsQb4x6",
    "groups": [],
    "id": "289383d9-ff3a-48bb-99ea-3048762267c7",
    "locale": "en-US",
    "meta": {
        "resourceType": "User"
    },
    "name": {
        "familyName": "Kenobi",
        "givenName": "Obi-Wan",
        "middleName": null
    },
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "obi-wan.kenobi@iamciam.dev"
}
```

- Now If you navigate to your local **scim** database and run `select * from users;`, you should see the assigned user in the database.

![SCIM_8](https://i.imgur.com/Kno7Jy7.png)

### Unassign a User

Let's go ahead and unassign the user we just assigned in Okta. Under **Assignments**, click the **X** next to the user we just assigned:

![SCIM_9](https://i.imgur.com/M3UzFX4.png)

The request/response should look like this:
 - *Okta Request:*
```
PATCH /scim/v2/Users/289383d9-ff3a-48bb-99ea-3048762267c7
```
```
{
    "schemas": [
        "urn:ietf:params:scim:api:messages:2.0:PatchOp"
    ],
    "Operations": [
        {
            "op": "replace",
            "value": {
                "active": false
            }
        }
    ]
}
```
 - *SCIM Server Response:*
```
204 NO CONTENT
```

- And sure enough, if you run that same `select * from users;` in the scim db, you will see the user is now `active: false`:

![SCIM_10](https://i.imgur.com/OjS9piE.png)

### Assign a Group
Under the **Assignments** tab, click **Assign** -> **Assign to Groups**. I assigned group **Droids**. Note that this acts similar to the above assigning of individual users. Okta will iterate through the group membership and create them in the external SCIM server - but the Group itself won't be made. That is done in **Push Groups** which we will handle later. Here is a look at the requests and responses from Okta to my SCIM server:

 - *Okta Request:*
```
GET /scim/v2/Users?filter=userName%20eq%20%22c-3po%40iamciam.dev%22&startIndex=1&count=100
```
 - *SCIM Server Response:*
```
200 OK
```
```
{
    "Resources": [],
    "itemsPerPage": 0,
    "schemas": [
        "urn:ietf:params:scim:api:messages:2.0:ListResponse"
    ],
    "startIndex": 1,
    "totalResults": 0
}
```
 - *Okta Request:*
```
POST /scim/v2/Users
```
```
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "c-3po@iamciam.dev",
    "name": {
        "givenName": "C",
        "familyName": "3PO"
    },
    "emails": [
        {
            "primary": true,
            "value": "c-3po@iamciam.dev",
            "type": "work"
        }
    ],
    "displayName": "C 3PO",
    "locale": "en-US",
    "externalId": "00usemeo53HWWPYy14x6",
    "groups": [],
    "password": "wA&&cprB",
    "active": true
}
```
 - *SCIM Server Response:*
```
201 CREATED
```
```
{
    "active": true,
    "displayName": "C 3PO",
    "emails": [
        {
            "primary": true,
            "type": "work",
            "value": "c-3po@iamciam.dev"
        }
    ],
    "externalId": "00usemeo53HWWPYy14x6",
    "groups": [],
    "id": "478c4cf4-1aa3-41a1-93da-4c154c5955e0",
    "locale": "en-US",
    "meta": {
        "resourceType": "User"
    },
    "name": {
        "familyName": "3PO",
        "givenName": "C",
        "middleName": null
    },
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "c-3po@iamciam.dev"
}
```

- There are 3 members in my **Droids** group: C-3P0, R2-D2 and BB-8. The above req/resp chain repeats for the other 2 users. Now when I run `select * from users;` in the scim database I can see the assigned users:

![SCIM_11](https://i.imgur.com/qw5bnQ9.png)

### Push Groups

As mentioned above, assigning a group in Okta only adds the users to my SCIM server, not the Group itself. In order to add the Group, tab over to **Push Groups** in your Okta app integration. Click **Push Groups** -> **Find groups by name** and select the group you assigned above. **Make sure Push group memberships immediately** is checked and click **Save**. Two calls should be made by Okta - one creating the group itself and another updating the group membership. Here are the requests and responses for my Droids group push:

 - *Okta Request:*
```
POST /scim/v2/Groups
```
```
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:Group"
    ],
    "displayName": "Droids",
    "members": [
        {
            "value": "cc2573a1-e1c4-491d-a3db-ae4a2e078d61",
            "display": "bb-8@iamciam.dev"
        },
        {
            "value": "478c4cf4-1aa3-41a1-93da-4c154c5955e0",
            "display": "c-3po@iamciam.dev"
        },
        {
            "value": "34e14e58-f0fe-4e9d-aeb2-0da25fa6626d",
            "display": "r2-d2@iamciam.dev"
        }
    ]
```
 - *SCIM Server Response:*
```
201 CREATED
```
```
{
    "displayName": "Droids",
    "id": "b446521a-a65b-4c0b-a5ee-0a15e8e3e908",
    "members": [],
    "meta": {
        "resourceType": "Group"
    },
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:Group"
    ]
}
```
```
POST /scim/v2/Groups
```
 - *Okta Request:*
```
PATCH /scim/v2/Groups/b446521a-a65b-4c0b-a5ee-0a15e8e3e908
```
```
{
    "schemas": [
        "urn:ietf:params:scim:api:messages:2.0:PatchOp"
    ],
    "Operations": [
        {
            "op": "add",
            "path": "members",
            "value": [
                {
                    "value": "cc2573a1-e1c4-491d-a3db-ae4a2e078d61",
                    "display": "bb-8@iamciam.dev"
                },
                {
                    "value": "478c4cf4-1aa3-41a1-93da-4c154c5955e0",
                    "display": "c-3po@iamciam.dev"
                },
                {
                    "value": "34e14e58-f0fe-4e9d-aeb2-0da25fa6626d",
                    "display": "r2-d2@iamciam.dev"
                }
            ]
        }
    ]
}
```
 - *SCIM Server Response:*
 ```
 200 OK
 ```
 ```
 {
    "displayName": "Droids",
    "id": "b446521a-a65b-4c0b-a5ee-0a15e8e3e908",
    "members": [
        {
            "display": "c-3po@iamciam.dev",
            "value": "478c4cf4-1aa3-41a1-93da-4c154c5955e0"
        },
        {
            "display": "bb-8@iamciam.dev",
            "value": "cc2573a1-e1c4-491d-a3db-ae4a2e078d61"
        },
        {
            "display": "r2-d2@iamciam.dev",
            "value": "34e14e58-f0fe-4e9d-aeb2-0da25fa6626d"
        }
    ],
    "meta": {
        "resourceType": "Group"
    },
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:Group"
    ]
}
 ```
 
 - And with this, you should now see your group in the scim database after running `select * from groups;`:
 
![SCIM_12](https://i.imgur.com/DWLnWEl.png)
 
 As well as a link table which displays the many-to-many relationship between the users and groups. You can see this with `select * from link;`:
 
 ![SCIM_13](https://i.imgur.com/4V0NAlc.png)
 -


This covers the basic functionality of SCIM integration from Okta. Other use-cases haven't been tested extensively and may need tweaking over time.
