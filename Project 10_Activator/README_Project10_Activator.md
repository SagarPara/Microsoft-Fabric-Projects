# Project 10 — Use Activator in Microsoft Fabric

**Workspace:** `Project 10_Activator`
**Activator:** `Contoso Shipping Activator`
**Object created:** `Redmond Packages`
**Rule created:** `Medicine temp out of range` (Running)
**Dataset:** Activator sample data — Contoso shipping/package delivery events
**Reference:** [MS Learn Lab — Use Activator in Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/11-data-activator.html)

---

## Overview

**Activator** is a Microsoft Fabric feature in the Real-Time Intelligence workload that lets you monitor streaming data and automatically trigger actions when specific conditions are met — without writing any code. Think of it as a rule engine sitting on top of your event data.

**Scenario for this lab:**
You are a data analyst for Contoso, a company that ships products to multiple cities. You are responsible for shipments to **Redmond**. One category of products is **medical prescriptions** that must remain refrigerated during transit. You need to create an alert rule that sends an email to the shipping department if the temperature of a **refrigerated medicine package** going to **Redmond** goes above a set threshold.

**Three core concepts in Activator:**

```
[Eventstream / data source]
        ↓
[Object]         ← a "thing" being monitored (e.g., a Package)
    ↓
[Property]       ← an attribute of that object (e.g., Temperature)
    ↓
[Rule]           ← a condition + filter + action on that property
    ↓
[Action]         ← what happens when the rule fires (email, Teams, etc.)
```

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 10_Activator`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `Contoso Shipping Activator` | Activator | Main Activator item with sample data, objects, and rules |

---

## Step 1 — Create the Activator

1. From the left menu, click **Create** → Under *Real-Time Intelligence*, select **Activator**.
   > If **Create** is not pinned, select **…** first.
2. A new Activator is created with a default timestamped name.
3. Click the dropdown next to the name (top-left) and rename it to `Contoso Shipping Activator`.
4. On the Activator home screen, click **Try sample** to load the sample shipping dataset.

After a moment, the **Explorer** pane on the left will populate with objects, properties, and pre-built rules from the sample data.

---

## Step 2 — Explore the Sample Data and Pre-Built Rules

### Explore the Package Delivery Events Stream

1. In the **Explorer** pane, scroll down and select **Package delivery events**.
2. The **Event details** live table shows real-time streaming events — each row is an incoming delivery event with fields like PackageId, City, Temperature, SpecialCare, ColdChainType, etc.

---

### Explore a Pre-Built Rule — `Too hot for medicine`

1. In the Explorer, under **Temperature (°C)**, select **Too hot for medicine**.
2. The **Definition** pane on the right shows how it works:
   - **Monitor section:** Attribute = Temperature (from the event data)
   - **Condition section:** Temperature is higher than 20°C
   - **Property filter:** SpecialCare = Medicine (only applies to medicine packages)
   - **Action section:** Sends a Teams message when triggered

3. Select your preferred action type → Click **Send me a test action** to receive a test notification.

> The Explorer also shows other pre-built rules: **Too cold for plants**, **Package created**, **Express shipping requested** — all demonstrating different trigger patterns.

---

## Step 3 — Create a New Object: `Redmond Packages`

The sample already has a **Package** object, but for this exercise we create a more targeted one — scoped to packages shipped specifically to **Redmond**.

1. Select the **Package delivery events** stream in the Explorer.
2. Click **New object** on the ribbon.
3. In the **Build object** pane, fill in:

   | Setting | Value |
   |---------|-------|
   | Object name | `Redmond Packages` |
   | Unique Identifier | `PackageId` |
   | Properties | `City`, `ColdChainType`, `SpecialCare`, `Temperature` |

4. Click **Create**.

The `Redmond Packages` object now appears in the Explorer with its four properties listed:
- **City** — delivery destination
- **ColdChainType** — e.g., Refrigerated, Ambient
- **PackageId** — unique identifier
- **SpecialCare** — e.g., Medicine, Plants

---

## Step 4 — Create the Rule: `Medicine temp out of range`

Now we create an alert rule on the `Redmond Packages` object that:
- Monitors **Temperature**
- Fires when temperature **increases above** the threshold
- Only for packages going to **Redmond**
- Only for packages containing **Medicine**
- Only for **Refrigerated** packages
- Sends an **email** alert

---

### Step 4a — Define the Condition

1. In the Explorer, select the **Temperature** property under `Redmond Packages`.
2. Click **New Rule** on the ribbon.
3. In the **Create rule** pane, enter:

   | Setting | Value |
   |---------|-------|
   | Condition | Increases above |
   | Value | `100` *(threshold value used in this exercise)* |
   | Occurrence | Every time the condition is met |
   | Action | Send me an email |

4. Click **Create**.
5. The rule is created with the default name *Temperature alert* — click the **pencil icon** next to the name and rename it to `Medicine temp out of range`.

> **Note:** The MS Lab guide uses a threshold of 20°C. In this exercise the value used is **100** — adjust based on your testing requirements. For production refrigerated medicine shipments, the acceptable range is typically **33°F to 41°F (0.5°C to 5°C)**.

---

### Step 4b — Add Property Filters

Now narrow the rule down with three filters so it only fires for the right packages.

In the **Definition** pane, expand **Property filter** and add the following:

**Filter 1 — City filter:**

| Setting | Value |
|---------|-------|
| Attribute | City |
| Operation | Is equal to |
| Value | Redmond |

Click **Add filter**.

**Filter 2 — Special care filter:**

| Setting | Value |
|---------|-------|
| Attribute | SpecialCare |
| Operation | Is equal to |
| Value | Medicine |

Click **Add filter**.

**Filter 3 — Cold chain filter:**

| Setting | Value |
|---------|-------|
| Attribute | ColdChainType |
| Operation | Is equal to |
| Value | Refrigerated |

> With all three filters, the rule will only fire when ALL conditions are true simultaneously: the package is going to Redmond **AND** contains Medicine **AND** is tagged as Refrigerated.

---

### Step 4c — Configure the Email Action

In the **Action** section, set:

| Setting | Value |
|---------|-------|
| Type | Email |
| To | Your current user account (default) |
| Subject | `Redmond Medical Package outside acceptable temperature range` |
| Headline | `Temperature too high` |
| Context | Select the **Temperature** property from the checkbox list |

Click **Save and start**.

The rule is now **Running** and actively monitoring the stream. In the dashboard you can see:
- **Instance:** 5 of 118 IDs (monitoring 5 packages out of 118 currently tracked)
- **Time:** Last 2 hours
- **Monitor chart:** Shows Temperature readings across multiple IOT device IDs (IOT-DF6262505, IOT-F2625055, IOT-PA4476128, IOT-R5581447, IOTR-MDF4791430)

---

## Step 5 — Verify the Rule is Running

Once the rule is saved and started:
- The rule status shows **Running** (green badge)
- The **Live feed** tab shows events that matched the conditions in real-time
- The **Analytics** tab shows aggregate stats about how often the rule fired
- The **History** tab shows a log of past trigger events

> The sample data generates random values in the background. If no data appears in the graph immediately, wait a few minutes and refresh. The chart shows whether temperature readings are trending toward or past the trigger threshold.

To stop the rule: click **Stop** on the ribbon.
To test it manually: click **Send me a test action** before starting.

---

## Activator Structure — Explorer Breakdown

The Explorer pane in `Contoso Shipping Activator` is organised as follows:

```
Contoso Shipping Activator
├── PackageId                          ← identifier property
├── Recipient email
├── Special care contents
├── Temperature (°C)                   ← pre-built property
│   ├── Too cold for plants            ← pre-built rule
│   └── Too hot for medicine           ← pre-built rule
├── Delivery events
│   └── Package created                ← pre-built rule
├── Redmond Packages                   ← NEW object (created in this lab)
│   ├── City
│   ├── ColdChainType
│   ├── PackageId
│   ├── SpecialCare
│   └── Temperature
│       └── Medicine temp out of range ← NEW rule (created in this lab)
├── Package delivery events            ← eventstream
└── Express shipping requested         ← pre-built rule
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Activator** | A Fabric item that monitors event data and triggers actions based on rules |
| **Object** | A "thing" being tracked — created from an eventstream using a unique identifier (e.g., PackageId) |
| **Property** | An attribute of the object to monitor (e.g., Temperature, City) |
| **Rule** | A condition applied to a property + filters + an action to take when triggered |
| **Property filter** | Narrows a rule to a subset of objects (e.g., only Redmond packages) |
| **Action** | What happens when the rule fires — Email, Teams message, etc. |
| **Occurrence** | How often the rule fires — "Every time" vs "Only once" vs "On change" |

---

## Summary

| Step | Action | Tool |
|------|--------|------|
| 1 | Create Activator and load sample data | Fabric Activator — Try sample |
| 2 | Explore Package delivery events stream and pre-built rules | Activator Explorer |
| 3 | Create new object `Redmond Packages` with 4 properties | New object |
| 4a | Create rule `Medicine temp out of range` — Temperature increases above 100 | New rule |
| 4b | Add 3 property filters: City = Redmond, SpecialCare = Medicine, ColdChainType = Refrigerated | Property filters |
| 4c | Configure email action with subject, headline, temperature context | Action section |
| 5 | Start rule and verify via Live feed / History tabs | Activator Rules view |

---

## Notes

- **Activator uses no-code rule building** — all condition and filter logic is configured via dropdowns in the Definition pane, with no KQL or Python needed.
- The **unique identifier** set during object creation (`PackageId` in this case) is what Activator uses to track each instance separately — the "5 of 118 IDs" shown in the screenshot means 5 specific packages out of 118 total are currently being evaluated against the rule.
- **"Every time the condition is met"** means the email fires each time a new data point crosses the threshold — not just once when it first crosses. Choose **"Only the first time"** if you want a one-shot alert.
- The **Property filter** section is AND-based — all filters must be true simultaneously for the rule to fire. This is important when combining City + SpecialCare + ColdChainType filters.
- The **sample data generates random values** in the background — this means the rule may or may not fire immediately depending on whether the generated temperature values happen to cross the threshold. This is expected behaviour.
- In a production setup, instead of sample data, the Activator would be connected to a real eventstream from IoT sensors, an Eventstream item, or a Power BI report.

---

*Documented by: Sagar Parab | Workspace: Project 10_Activator | Last updated: March 2026*
