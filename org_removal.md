# Cleaning the Postgres database

## Get the user

Start with: account email address.

`example@email.com`

Get user id.

`SELECT id FROM auth_user WHERE email='example@email.com'`

(example user id will be 10)

Get user organizations. There can be multiple.

`SELECT * FROM questionnaire_organization_users WHERE user_id=10`

(example organization id will be 20)


## Get answers

Next, delete the answers. Answers are stored in questionnaire_answer, but some may also have rows in questionnaire_answer_options. You will need to find which answers may be in the latter table before deleting.

`SELECT * FROM questionnaire_answer WHERE organization_id=20 AND text IS NULL AND integer IS NULL AND yesno IS NULL`

Not all of these answers will also have a row in questionnaire_answer_option, but some will. Compile the ids of these answers (listed in the id column) into a list seperated by comma. Replace the list in the example, with your own.

`DELETE FROM questionnaire_answer_options WHERE answer_id IN (31, 32, 33, 34)`

Now you can go back and delete the rows from questionnaire_answer.

`DELETE FROM questionnaire_answer WHERE organization_id=20`

Go through these steps for all of the organizations you want to remove. If you are only removing some organizations and not an entire user, then skip [Deleting a user](##deleting-a-user) and go to [Deleting an organization](##deleting-an-organization).


## Deleting a user

Delete the user id from the questionnaire_activeorganization table.

`DELETE FROM questionnaire_activeorganization WHERE user_id = 10`

Delete all organizations associated with the user_id from questionnaire_organization_users.

`DELETE FROM questionnaire_organization_users WHERE user_id = 10`

Delete from token list. Once you do this step, the user will be logged out and will no longer be able to access the questionnaire.

`DELETE FROM authtoken_token WHERE user_id=10`

Finally, you can remove the user from auth_user.

`DELETE FROM auth_user WHERE id=10`


## Deleting an organization

Delete the organization id from the questionnaire_activeorganization table.

`DELETE FROM questionnaire_activeorganization WHERE organization_id = 20`

Delete the organization from the questionnaire_organization_users table.

`DELETE FROM questionnaire_organization_users WHERE organization_id = 20`


# Cleaning the triplestore

When it comes to deleting triples from the triplestore, you will simply need to clear all graphs associated with the organization. Clear graphs is done with the following format:

`CLEAR GRAPH <https://link.to/graph>`

Please note that you must keep the angeled brackets around the graph IRI.

Each question has a unique graph so you may end up having to delete multiple graphs, and you can not list multiple graphs to delete in one query, so this may end up taking some time unless you programatically make a more efficient means of deleting multiple graphs.

To find all of the graphs associated with your organization, you can use the following query, with ### replaced by your organization id:

```
SELECT distinct ?g
WHERE {
  GRAPH ?g { ?s ?p ?o }
  FILTER regex(str(?g), "https://cafe-trauma.com/cafe/organization/###/", "i" )
}
```