# Overview

Content can be excluded _globally_ or _per user_ configuration.

Exclusion can happen during _indexing_, _querying_ or _post-retrieval_.

All of the approaches have their own pros and cons. Can be implemented independently or in conjunction.

|                      |                                                                                                         |                                                                       |
|----------------------|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| **Application time** | **Global exclusion**                                                                                    | **Per user exclusion**                                                |
| Indexing             | Excluded content is not indexed                                                                         | N/A                                                                   |
| Query                | New index field excludedGlobally used by index query.                                                   | New index field excludedByUser (list of strings) used by index query. |
| Post-retrieval       | Apply post-retrieval results filtering by applying a combination of global and per-user exclusion rules |                                                                       |

# Configuration

Exclusion configuration is independent of application time.

Configuration structure (expand):

Global exclusion:

`Id            - PK, identity ClientId      - FK -> Clients.Id TargetKind    - Document/PageSubtree/ConfluenceFolder DocumentUrl   - Blob storage URL DocumentTitle - Display name InactiveDate  - Nullable, soft delete`

Per-user exclusion:

`Id            - PK, identity ClientId      - FK -> Clients.Id UserId        - FK -> Users.Id TargetKind    - Document/PageSubtree/ConfluenceFolder DocumentUrl   - Blob storage URL DocumentTitle - Display name InactiveDate  - Nullable, soft delete`

Note: ClientId+UserId could be replaced by UserClientId (FK -> UserClients.Id).

# Application

Exclusion can be applied during **indexing** or **post-retrieval**. Each approach has it’s own pros and cons, can implement either one or both.

Both approaches can coexist to mitigate downsides of each.

|                                   |                     |                                                              |                                                                                                                |
|-----------------------------------|---------------------|--------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
|                                   | **Index exclusion** | **Query exclusion**                                          | **Post-retrieval exclusion**                                                                                   |
| Overview                          |                     |                                                              |                                                                                                                |
| General considerations            |                     | Fits overall design naturally, but application time is slow. | Immediate application, but excluded content will eat up search results slots (top N results become N-excluded) |
| Effect                            |                     | After next indexing (~30 min or less if launched manually)   | Immediate                                                                                                      |
| Debug complexity                  |                     | Medium                                                       | Low                                                                                                            |
| Per-user configuration complexity |                     | Medium                                                       | Low                                                                                                            |

# Folder exclusion (parent-child relation)

Excluded folder should automatically exclude all members of the folder.

When page is moved it should automatically reapply exclusion rules.

Cases:

- Generic blob storage folder
    
- Confluence folder
    
- Confluence parent page
    

## Generic blob item exclusion

## Confluence page and folder exclusion

Option matrix: