# Fixing questions

Sometimes the wording on a question or the questionnaire logic needs to be fixed. This is a fairly simple process.

Access the psql database being used by your CAFE server. Look up and verify the id of the question you wish to modify.

Modify using the following format:

`UPDATE questionnaire_question SET text='This is the new wording on my question?' WHERE id=999;`

The two columns you are likely to modify are **text** for the wording of the question and **depends_string** for the questionnaire logic.

While we haven't been using it, for the sake of thoroughness, you should edit **test.json** in the cafe-server repository.

Finally, it would be prudent to update the **questionnaire_question.csv** file that was used to fill in the question table when the server was first set up. This file is not in one of these repositories. We have ours stored on the share drive in the CAFE folder.
