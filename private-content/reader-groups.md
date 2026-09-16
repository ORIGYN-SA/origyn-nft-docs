---
icon: users-rectangle
---

# Reader Groups

{% hint style="warning" %}
**Private content works only through the HTTP API for now.** Reader groups only matter for private content, so manage them through the endpoints on this page with your [API key](../rest-api/api-keys.md).
{% endhint %}

A reader group is a named list of principals inside an [organization](../minting-studio/organizations.md). Name the group in a template item's `readers`, and every member of the group can read that item on every certificate of the organization's collections. See [Private Content](overview.md#marking-fields-private) for how templates reference groups.

Reader groups are for people **outside** your organization: an insurer, an auditor, a service partner. Members of the organization already read all private content and do not need a group.

## Who can manage groups

| Action | Required role |
| ------ | ------------- |
| List groups | Any active member of the organization |
| Create or delete a group, add or remove members | **Owner** or **Admin** |

A suspended organization cannot change its groups.

{% hint style="info" %}
**Group membership is public.** Groups are recorded on chain, and anyone can see who belongs to which group. Only the certificate content they unlock is private.
{% endhint %}

## Limits

| Limit | Value |
| ----- | ----- |
| Groups per organization | 32 |
| Members per group | 256 |
| `group_id` | 1 to 32 characters of `a-z`, `0-9` and `-`. `owner` and `members` are reserved. |
| `name` | 1 to 64 bytes |

`group_id` is what templates and certificates refer to, and it cannot be changed. `name` is a display label.

## Create a group

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/orgs/12/groups \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "group_id": "insurer", "name": "Insurer" }'
```

```json
{
  "group": {
    "group_id": "insurer",
    "name": "Insurer",
    "members": [],
    "member_count": 0,
    "created_by": "<principal>",
    "created_at": "2026-09-16T10:30:00Z"
  }
}
```

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## List groups

Returns every group in the organization with its members, read live from the canister.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Add a member

The body names the principal to add. Adding someone who is already a member succeeds and changes nothing. The anonymous principal cannot be added.

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/orgs/12/groups/insurer/members \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "principal": "<reader_principal>" }'
```

Answers `204` with no body.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}/members" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Remove a member

Answers `204`. Removing a principal who is not a member also succeeds.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}/members/{principal}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Delete a group

Deletes the group and every membership in it. Answers `204`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="warning" %}
**Changes take effect with a short delay.** Access follows the gateway's read service, which normally picks up a change within a minute. `GET /orgs/{org_id}/groups` reads the canister directly, so it shows a removal before the removed reader actually loses access.

Deleting a group does not remove its `group_id` from templates or certificates. If you create a group with the same id later, its new members get that access. Revocation never recalls what a reader has already downloaded; see [Revoking access](overview.md#revoking-access).
{% endhint %}

## Errors

| Status | `error` | Meaning |
| ------ | ------- | ------- |
| `400` | `invalid_group_id` | `group_id` breaks the format above, or is reserved |
| `400` | `invalid_name` | `name` is empty or longer than 64 bytes |
| `400` | `invalid_principal`, `anonymous_principal` | The member principal is malformed or anonymous |
| `403` | `not_a_member` | You are not an active member of this organization, or it does not exist |
| `403` | `insufficient_role` | Your role cannot manage groups |
| `403` | `org_suspended` | The organization is suspended |
| `403` | `not_authorized` | Your principal has not authorized the gateway yet (see [Obtaining an API Key](../rest-api/api-keys.md)) |
| `404` | `group_not_found` | No group with this id |
| `409` | `group_exists` | A group with this id already exists |
| `409` | `too_many_groups`, `too_many_members` | A limit above was reached |
