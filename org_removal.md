Start with: account email address.
`example@email.com`

Get user id.
`SELECT id FROM auth_user WHERE email='example@email.com`
(example user id will be 10)

Delete from token list.
`SELECT * FROM authtoken_token WHERE user_id=10`

Get user organizations. There can be multiple.
`SELECT * FROM questionnaire_organization_users WHERE user_id=10`
(example organization id will be 20)

Find questionnaire answers for organization.
`SELECT * from questionnaire)answer WHERE organization_id=20`

Some answers may be further linked to answer options.
`SELECT * FROM questionnaire_answer WHERE organization_id=20 AND text IS NULL AND integer IS NULL AND yesno IS NULL`
Not all of these answers will have an answer option but some will.
`SELECT * FROM questionnaire_answer_options WHERE answer_id IN (<List of applicable answer ids seperated by comma>)`

The table questionnaire_statement contains the format for triples written for ech question.