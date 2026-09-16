# dbrx-git-folders-dea
Repository for databrick data engineer associate course by Ramesh Retnaswamy

# David chu KC meeting
[Image]

A senior technical speaker provided a detailed walkthrough of the Garmin Messenger and inReach messaging system architecture to a colleague. The discussion covered the complex logic for counting messages (IM, Post, MO, MT), methods for identifying and linking users and devices, and the critical differences between querying aggregated data in Application Insights versus detailed records in Cosmos DB. The session concluded with best practices for retrieving user subscription information, offering a foundational understanding essential for data analysis and fraud prevention efforts.

------------
## Message Counting and Aggregation Logic

This section details the rules for counting and categorizing different types of messages (IM, Post, MO, MT) within the system. It clarifies how group chats and "post messages" are handled, explaining that a single user action can result in multiple message entries depending on the delivery path and message type, which directly impacts statistical reporting.

In a standard group chat, all messages share a single, common `conversation ID`. However, a special message type known as a "post message" is handled differently. Instead of being lumped together, a post message is treated as if the sender messaged each recipient separately, generating a unique `conversation ID` for each individual post. Although a post message appears as only one row in the `inReach message` table, the system's backend logic splits it into multiple distinct conversations.

The system distinguishes between Message Originating (MO) and Message Terminating (MT) counts, which affects statistical reporting. For instance, there is no true "MT IM" message; the receiving party simply fetches the same original message sent by the originator. When a user sends a message to another user who has two inReach devices, the system creates two separate inReach messages, but the MO count still registers it as a single originating message. Statistics are further refined by transport method; messages sent from an inReach device are excluded from phone app stats, and vice versa.

Messages sent from Garmin OS watches, which use a phone's internet connection, are categorized via a "message from device" query that differentiates by device type. A notable mechanism is the "co-app push" for watches, where the system first attempts to deliver a message notification over LTE. If the watch is offline, it can then retrieve the message via satellite. This single MT event is represented as one count in summary stats but creates two entries in the underlying data: one "Skilo" message and one "GCS coap push" message. A critical limitation is that Application Insights, the primary source for these stats, contains only aggregated data, making it impossible to retrieve individual `message ID`s or `conversation ID`s.

------------
## User and Device Identification and Data Structure

This section explains how different user types (Garmin account holders, Messenger-only users) and devices (inReach, Garmin OS watches, mobile apps) are identified and linked within the database. It covers the roles of the `user addresses`, `user instances`, and `devices` tables, the significance of the `SSO GUID` (Garmin GUID), and the distinction between a paired IMEI and the current array-based system for multiple devices.

The system differentiates users based on whether they have a full Garmin account. Users with a Garmin account are linked via a `Garmin GUID`, which is stored in the `user addresses` table under the legacy field name `SSO GUID`. In contrast, Messenger-only users register with just a phone number and do not have a Garmin account or an associated `SSO GUID`. Linking a Garmin account is an optional step for Messenger-only users, but it is an automatic part of the process for those who register an inReach device.

The `user instances` table serves as a central registry that links each user to their associated devices, which can include inReach units, Garmin OS watches (like a Forerunner 965 or Fenix 7/8), and mobile app instances. A user can have multiple app instances (e.g., on a phone and an iPad) tracked in this table. Even users without any Garmin hardware will appear in the `user instances` table with an "app instance" if they use the Messenger app. The system has evolved from a `paired inReach IMEI` field, which is now deprecated, to an array-based structure that supports multiple inReach devices per account.

------------
## Data Sources and Query Limitations: App Insights vs. Cosmos DB

This section contrasts the two primary data sources for system metrics: the aggregated, non-granular data in Application Insights versus the detailed, but ephemeral, raw data in Cosmos DB. It explains why App Insights was adopted for performance and sustainability but highlights its limitations for answering specific, historical queries about individual users or devices, which are purged from Cosmos DB.

Initially, metrics were generated by querying the Cosmos DB directly, but this practice proved unsustainable, as it was "killing" the database and causing severe performance issues. To resolve this, the team transitioned to using Application Insights, a more performant solution for publishing and querying telemetry logs. However, this shift introduced significant limitations.

The data in Application Insights is strictly aggregated and does not contain granular, message-level details. Consequently, it is impossible to retrieve individual `message ID`s or `conversation ID`s from this source. This makes it unfeasible to answer specific historical questions, such as "How many messages did a specific device send three months ago?" The challenge is compounded by the data retention policy for Cosmos DB, where inReach message data is purged after approximately 14 days. While App Insights is functional for high-level metrics, its lack of flexibility makes long-term data correlation and detailed individual analysis difficult.

------------
## Subscription and Plan Information Retrieval

This section outlines the methods for determining a user's subscription plan, which is crucial for features like the "Suspend with SOS" plan. It discusses looking up the `plan skew` in the `SP devices` table, its limitations due to not being backfilled, and recommends using the IMEI or Garmin GUID to query an external API (Turbo Checker) as the most reliable source of truth for subscription status.

The `plan skew` field in the `SP devices` table was introduced to support a new "Suspend with SOS" plan, which restricts users to only sending SOS messages. However, this field is not a completely reliable source for subscription information. Because the data was not backfilled for devices activated before the feature's implementation in the summer of 2023, many older records have a `null` value for `plan skew`. While a populated `plan skew` likely indicates an activated device, a `null` value is ambiguousâit could mean the device is not activated or simply that its data was never backfilled.

Given these limitations, the most reliable method for verifying a device's subscription status is to query an external API known as "Turbo Checker." This system serves as the definitive source of truth for subscription information. The API call can be made using either the device's `IMEI` or the user's `Garmin GUID` (`SSO GUID`), depending on which identifier is more readily available.

------------
## System Monitoring: Fraud Detection and Automated Testing

This section describes the mechanisms in place for monitoring system health and preventing abuse. It explains the purpose of a fraud detection query that tracks "new conversation initiated" events to flag anomalous activity, such as phishing attacks. It also mentions the "traffic checker," an automated test that continuously injects messages to ensure system functionality.

The development of a fraud detection system was prompted by an incident where a malicious actor used Postman to hit the system's API and send phishing messages to tens of thousands of phone numbers. To prevent future abuse, a query was created to monitor the frequency of "new conversation initiated" events in Application Insights. If a user starts an anomalous number of new conversations in a single day (e.g., 50 new conversations), an alert is triggered.

In addition to fraud monitoring, an automated testing tool called the "traffic checker" runs continuously in both the test and production environments. This tool injects messages into the system and verifies they reach their destination, ensuring end-to-end functionality. This, along with manual testing by team members during the workday, accounts for the constant message activity observable in system logs.

------------
## Action Items

**@Speaker 2**
- [ ] Investigate the `inReach message` table for an "is post" parameter to identify post message types - [TBD]
- [ ] Schedule a follow-up meeting for further technical questions - [TBD]
- [ ] Take a photograph of the whiteboard diagram for future reference - [TBD]
