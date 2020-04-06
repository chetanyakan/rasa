:desc: Set up a CI/CD pipeline to ensure that iterative improvements are tested and deployed with minimum manual effort

.. _setting-up-ci-cd:

Setting up CI/CD
================

Developing a bot looks a little different than developing a traditional
software application, but that doesn't mean you should abandon software
development best practices. Setting up CI/CD ensures that incremental updates
to your bot are improving it, not harming it.

.. contents::
   :local:
   :depth: 2


Overview
--------

Continous Integration (CI) is the practice of merging in code changes
frequently, and automatically testing changes as they are committed. Continuous
Deployment (CD) means automatically deploying integrated changes to a staging
or production environment. Together, they allow for more frequent improvements
to your assistant with less manual effort expended on testing and deployment.

This guide will cover **what** should go in a CI/CD pipeline, specific to a
Rasa project. **How** you implement that pipeline is up to you. There are many
CI/CD tools out there, and you might already have a preferred one. Many Git
repository hosting services like Github, Gitlab, and Bitbucket also provide
their own CI/CD tools that you can make use of. 

Continuous Integration
----------------------

Assistants are best improved with frequent, `incremental updates
<https://rasa.com/docs/rasa-x/user-guide/improve-assistant/>`_.
No matter how small a change is, you want to be sure that it doesn't introduce
new problems or negatively impact the performance of your assistant. Most tests are
quick enough to run on every change. However, you can set some more
resource-intentsive tests to run only when certain files have been changed or
when a certain tag is present.

CI checks usually run on commit, or on merge/pull request.

.. contents::
   :local:

Validate Data
#############

:ref:`Data validation <validate-files>` verifies that there are no mistakes or
major inconsistencies in your domain file, NLU data, or story data. 

.. code-block:: bash

   rasa data validate --fail-on-warnings

If ``rasa
data validate`` results in errors, training a model will also fail. By
including the ``--fail-on-warnings`` flag, the command will also fail on
warnings about problems that won't prevent training a model, but might indicate
messy data, such as actions listed in the domain that aren't used in any
stories.

Validate Stories
################

:ref:`Story validation <test-story-files-for-conflicts>` checks if you have any
stories where different bot actions follow from the same dialogue history.

.. code-block:: bash

   rasa data validate --fail-on-warnings --max-history 5

Conflicts between stories will prevent Rasa from learning the correct pattern.
You can run story validation by passing the ``--max-history`` flag to ``rasa
data validate``, either in a seperate check or as part of the data validation
check. ``--max-history`` should be set to the value of ``max_history`` in your
config for your memoization policy (which defaults to ``5``).

Train a Model
#############

.. code-block:: bash

   rasa train

Training a model verifies that your NLU pipeline and policy configurations are
valid and trainable, and it provides a model to user for test conversations. Training a model is also :ref:`part of the continuous deployment
process <uploading-a-model>`, as you'll need a model to upload to your server. 

End-to-End Testing
##################

Testing your trained model on :ref:`test conversations
<end-to-end-testing>` is the best way to have confidence in how your assistant
will act in certain situations. These stories, written in a modified story
format, allow you to provide entire conversations and test that, given this
user input, your model will behave in the expected manner. This is especially
important as you start introducing more complicated stories from user
conversations. End-to-end testing is only as thorough and accurate as the test
cases you write, so you should always update your test conversations
concurrently with your training stories.

.. code-block:: bash

   rasa test --stories tests/test_stories.md --fail-on-prediction-errors

To ensure the test will fail if any test conversation fails, add 
the ``--fail-on-prediction-errors`` flag:

Note: End-to-end testing does **not** execute your action code. You will need to
:ref:`test your action code <testing-action-code>` in a seperate step.

NLU Comparison
##############

If you've made significant changes to your NLU training data (such as adding or
splitting intents, or just adding/changing a lot of examples), you should run a
:ref:`full NLU evaluation <nlu-evaluation>`. You'll want to compare
the performance of the NLU model without your changes to an NLU model with your
changes. 

You can do this by running NLU testing in cross-validation mode:

.. code-block:: bash

   rasa test nlu --cross-validation

or by training a model on a training set and testing it on a test set. If you use the train-test
set approach, it is best to :ref:`shuffle and split your data <train-test-split>` using ``rasa data split`` every time, as
opposed to using a static NLU test set, which can easily become outdated. 

Since NLU comparison can be a fairly resource intensive test, you can set this
test to run only when a certain tag (e.g. "NLU testing required") is present,
or only when changes to NLU data or the NLU pipeline were made. 

.. _testing-action-code:

Testing Action Code
###################

The approach used to test your action code will depend on how it is
implemented. Whichever method of testing your code you choose, you should
include running those tests in your CI pipeline as well. For example, if you
connect to external API's it is recommended to write unit tests to ensure 
that those APIs respond as expected to common inputs.

Continuous Deployment
---------------------

To get improvements out to your users frequently, you need to automate as
much of the deployment process as possible. 

CD steps usually run on push or merge to a certain branch, once CI checks have
succeeded.

.. contents::
   :local:

.. _uploading-a-model:

Deploying your Rasa Model
#########################

You should already have a trained model from running end-to-end testing in your
CI pipeline. You can set up your pipeline to upload the trained model to your
Rasa server. If you're using Rasa X, you can also 
make an `API call <https://rasa.com/docs/rasa-x/api/rasa-x-http-api/#tag/Models/paths/~1projects~1{project_id}~1models~1{model}~1tags~1{tag}/put>`_ 
to tag the uploaded model as ``production`` (or whichever `deployment environment <https://rasa.com/docs/rasa-x/enterprise/deployment-environments/#>`_ you want
to deploy it to).

For Rasa X, this could look something like:

.. code-block:: bash

   curl -k -F "model=@models/my_model.tar.gz" "https://example.rasa.com/api/projects/default/models?api_token={your_api_token}"
   curl -X PUT "https://example.rasa.com/api/projects/default/models/my_model/tags/production"


However, if your update includes changes to both your model and your action
code, and these changes depend on each other in any way, you should **not**
automatically tag the model as ``production``. You will first need to build and
deploy your updated action server, so that the new model won't e.g. call
actions that don't exist in the pre-update action server.

Deploying your Action Server
############################

If you're using a containerized deployment of your action server, you can
automate `building a new image <https://rasa.com/docs/rasa/user-guide/docker/building-in-docker/#adding-custom-actions>`_, 
uploading it to an image repository, and deploying a new image tag for each
update to your action code. As noted above, you should be careful with
automatically deploying a new image tag to production if the action server
would be incompatible with the current production model.

Example CI/CD pipelines
-----------------------

As examples, see the CI/CD pipelines for 
`Sara <https://github.com/RasaHQ/rasa-demo/blob/master/.github/workflows/build_and_deploy.yml>`_,
the Rasa assistant that you can talk to on this website, and for
`Carbon Bot <https://github.com/RasaHQ/carbon-assistant/blob/master/.github/workflows/model_ci.yml>`_. 
Both use `Github Actions <https://github.com/features/actions>`_ as a CI/CD tool. These examples are far 
from the only ways to do it; in fact, if you have a CI/CD set up you'd like to share with the
Rasa community, please post on the `Rasa Forum <https://forum.rasa.com/>`_.
