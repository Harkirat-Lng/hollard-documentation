# Restricted User — Hollard API

A "restricted user" is a user the platform refuses to verify, shown the message *"We couldn't verify you. Please call us if you need help."* (HTTP 501). The logic lives in `cc.services/logic/Implementation/SecurityQuestionService.cs` and is driven by four config keys in `cc.api/appsettings.json`:

| Config key | Current value | Used by |
|---|---|---|
| `RestrictUnverifiedUsers` | `true` | `UserVerificationCheck` |
| `RestrictedSurname` | `BEKKER,ROBERTSON` | `UserVerificationCheck` |
| `RestrictedIdNumberPrefix` | `30-45` | `validateRestrictedIdNumberPrefix` |
| `RestrictedAnswersCount` | `4` | `validateRestrictedAnswers` |

There are two separate places a user gets restricted.

## 1. Bureau basic-info verification (registration)

Endpoint `User/ValidateUserBasicInfoWithBureauForHollard` (`cc.api/Controllers/UserController.cs`) sets `BasicInfoResponseModel.IsRestrictedUser`. If the bureau matches all three of surname, full name and phone number, the user is verified and never restricted. Only on a failed match does `UserVerificationCheck` run.

```mermaid
flowchart TD
    A["POST User/ValidateUserBasicInfoWithBureauForHollard"] --> B["Bureau basic-info lookup<br/>BasicInfoBureauServiceV2"]
    B --> C{"Surname AND Fullname AND Number<br/>all match at bureau?"}
    C -->|Yes| V["IsVerify = true<br/>IsRestrictedUser = false"]
    C -->|No| D["UserVerificationCheck"]
    D --> E{"Config RestrictUnverifiedUsers = true?<br/>currently true"}
    E -->|Yes| F{"Bureau response JsonString is null?"}
    F -->|Yes| R1["RESTRICTED<br/>NoBasicInfoLogForUserInDatabase"]
    F -->|No| G{"JsonString empty OR contains<br/>'no information found'?"}
    G -->|Yes| R2["RESTRICTED<br/>NullResponseFromBureau /<br/>NoInformationFoundAtBureau"]
    G -->|No| H
    E -->|No| H{"User surname in RestrictedSurname list?<br/>config: BEKKER, ROBERTSON"}
    H -->|Yes| R3["RESTRICTED<br/>RestrictedSurname"]
    H -->|No| OK["Not restricted<br/>IsVerify = false, proceeds to security questions"]

    style R1 fill:#f8d7da,stroke:#d9534f,color:#000
    style R2 fill:#f8d7da,stroke:#d9534f,color:#000
    style R3 fill:#f8d7da,stroke:#d9534f,color:#000
    style V fill:#d4edda,stroke:#28a745,color:#000
    style OK fill:#d4edda,stroke:#28a745,color:#000
```

## 2. Restricted-answers check (security questions)

When fetching security questions (`GetSecurityQuestionsPart1` / `GetSecurityQuestionsPart2`), `validateRestrictedAnswers` can block the user with HTTP 501.

```mermaid
flowchart TD
    A["GetSecurityQuestionsPart1 / Part2"] --> B{"ID number starts with prefix in<br/>RestrictedIdNumberPrefix range?<br/>config: 30-45, i.e. born 1930-1945"}
    B -->|No| OK["Not restricted — questions returned"]
    B -->|Yes| C["Count correct answers that start with<br/>'i do not' or 'i am not'"]
    C --> D{"count ≥ RestrictedAnswersCount?<br/>config: 4"}
    D -->|Yes| R["RESTRICTED — HTTP 501<br/>'We couldn't verify you.<br/>Please call us if you need help.'"]
    D -->|No| OK

    style R fill:#f8d7da,stroke:#d9534f,color:#000
    style OK fill:#d4edda,stroke:#28a745,color:#000
```

## Summary of the conditions

A user becomes **restricted** when any one of these holds:

1. **No bureau record** — bureau basic-info match failed and the bureau returned no data for them (null response, empty response, or "no information found"), while `RestrictUnverifiedUsers` is enabled.
2. **Blacklisted surname** — their surname is on the `RestrictedSurname` config list, checked after a failed bureau match regardless of the unverified-users flag.
3. **Elderly ID + evasive answers** — their SA ID number indicates birth year 1930–1945 (`RestrictedIdNumberPrefix`) **and** at least `RestrictedAnswersCount` (4) of their stored correct security-question answers are of the "I do not… / I am not…" type. This is a fraud heuristic: for high-risk identities, a profile whose correct answers are mostly "none of the above"-style is refused verification outright.

A user whose bureau match succeeds on all three fields is never restricted via paths 1 or 2 — those checks only run on a failed match. All restriction events are logged as `"User Restricted:"` with status code 501, so occurrences can be traced in Application Insights.