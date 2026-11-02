Fix these forge PR review comments (auto-route):

### agent comment 6569769336
Code fix, no spec change: ModelGateway() must not raise in __init__ when ANTHROPIC_API_KEY is unset. Raise MissingAPIKeyError from require_client() and require_skills_client(), as AC8 says. Keep the existing test.
