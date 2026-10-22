# CUCM 14 CIPC Caller-ID Transformation Lab

A practical, troubleshooting-led lab showing how to transform the calling-party identity from an originating Cisco IP Communicator (CIPC) phone in CUCM 14, using a dedicated Calling Party Transformation Pattern, Partition, and Calling Search Space.

> **Lab result:** CIPC DN 2102 called Cisco 7821 DN 2104 internally. Initially, the 7821 displayed 2102. The call began displaying the configured external E.164 identity after the originating CIPC was configured to use a dedicated Calling Party Transformation CSS in **Number Presentation Transformation → Caller ID for Calls from This Phone**.

## Contents

- [Scenario and goal](#scenario-and-goal)
- [Key concepts](#key-concepts)
- [Final working configuration](#final-working-configuration)
- [Step-by-step procedure](#step-by-step-procedure)
- [Validation and Dialed Number Analyzer](#validation-and-dialed-number-analyzer)
- [Troubleshooting: what did not work](#troubleshooting-what-did-not-work)
- [Scaling to more phones](#scaling-to-more-phones)
- [Change control and rollback](#change-control-and-rollback)
- [References](#references)

## Scenario and goal

The lab has CUCM 14 with these endpoints:

| Endpoint | Directory number | Role |
|---|---:|---|
| CIPC on PC2 | 2102 | Originating phone |
| Cisco 7821 desk phone | 2104 | Receiving phone |
| Other lab CIPC | 2103 and 1003 | Candidate extensions for a later expansion |

The internal test call is:

    CIPC 2102 → CUCM → Cisco 7821 2104

This is an internal CUCM call. It does not need to traverse CUBE or Twilio simply because a Twilio number is used as the desired caller identity.

For public documentation, the actual Twilio number is intentionally represented by:

    <YOUR_TWILIO_E164_DID>

Replace that placeholder in your private lab notes with the authorized number you use. Do not publish phone numbers, passwords, authentication secrets, private keys, or sensitive screenshots.

### Goal

For this lab, the receiving 7821 should display the configured E.164 identity when CIPC 2102 calls it. This is a **shared caller identity** test: the fixed mask replaces the matched calling number with the same configured DID. It does not give every extension a unique public number.

## Key concepts

CUCM has several similarly named CSS fields. They are not interchangeable.

| Field | Primary purpose |
|---|---|
| **Line Calling Search Space (Line CSS)** | Controls the dial-plan partitions the directory number can search for destinations. |
| **Device Calling Search Space (Device CSS)** | Controls the dial-plan partitions available through the phone device configuration. |
| **Calling Party Transformation CSS** | Makes calling-party transformation patterns in its selected partitions available to the relevant calling-party transformation process. |
| **Calling Party Transformation Pattern** | Matches a calling number and supplies transformation instructions, such as a replacement mask. |
| **Calling Party Transformation Mask** | Defines the transformed calling number for the matched pattern. |
| **Use Device Pool Calling Party Transformation CSS** | In the phone’s caller-ID transformation section, determines whether the phone inherits that CSS from the device pool instead of using the phone-level selection. |

The pattern and the CSS have to work together. Creating a transformation pattern alone does not make it available to every phone. A phone's ordinary Line CSS or Device CSS is not a substitute for the Calling Party Transformation CSS field.

## Final working configuration

The working arrangement was:

- Dedicated partition: **PT Caller ID Lab**
- Exact Calling Party Transformation Pattern: **2102**
- Calling Party Transformation Mask: **<YOUR_TWILIO_E164_DID>**
- Dedicated CSS: **CSS Caller ID Lab**
- CSS Caller ID Lab contains **PT Caller ID Lab only**
- On the originating CIPC 2102, **Use Device Pool Calling Party Transformation CSS** was unchecked in **Number Presentation Transformation → Caller ID for Calls from This Phone**
- The phone-level **Calling Party Transformation CSS** was set to **CSS Caller ID Lab**

After saving and applying the change, the CIPC restarted/re-registered. The internal call from 2102 to 2104 then presented the configured calling identity on the receiving 7821.

The exact values below document the lab method. Confirm the field labels against the CUCM 14 SU deployed in your environment.

## Step-by-step procedure

### 1. Create a dedicated partition

In CUCM Administration, navigate to:

    Call Routing → Class of Control → Partition

Create:

| Field | Value |
|---|---|
| Name | PT Caller ID Lab |
| Description | Dedicated calling-party transformation lab |

Keep this partition dedicated to transformation patterns. Do not populate it with ordinary directory numbers or unrelated route patterns.

### 2. Create the Calling Party Transformation Pattern

Navigate to:

    Call Routing → Transformation → Transformation Pattern → Calling Party Transformation Pattern

Choose **Add New**.

| Field | Lab value |
|---|---|
| Pattern | 2102 |
| Partition | PT Caller ID Lab |
| Description | CIPC 2102 caller-ID transformation test |
| Calling Party Transformation Mask | <YOUR_TWILIO_E164_DID> |
| Use Calling Party's External Phone Number Mask | Off |
| Prefix Digits | Leave blank |
| Calling Party Presentation | Allowed |
| Calling Party Number Type / Numbering Plan | Retain suitable defaults for the lab |

Save the pattern.

**Why leave “Use Calling Party's External Phone Number Mask” off?** This lab explicitly supplies the Calling Party Transformation Mask. The external phone number mask is a different mechanism based on the directory number's configured External Phone Number Mask. Do not enable the option just because the desired display is in E.164 format. If you later implement an organization-wide numbering plan, make a deliberate choice between explicit transformation masks and per-DN external masks.

### 3. Create a dedicated CSS

Navigate to:

    Call Routing → Class of Control → Calling Search Space

Create:

| Field | Value |
|---|---|
| Name | CSS Caller ID Lab |
| Selected partition | PT Caller ID Lab only |

Keep normal routing partitions out of this CSS. In the lab, PT HQ Internal, PT HQ Services, and PT HQ PSTN Test belong to ordinary dial-plan policy, not this dedicated transformation CSS.

The existing CSS HQ PSTN Test was returned to its normal routing partitions. The new CSS Caller ID Lab was kept separate.

### 4. Apply the CSS to the originating CIPC

Navigate to:

    Device → Phone → HQ-PC2-SIP-TCP-LAB

Open the CIPC device associated with DN 2102 and locate:

    Number Presentation Transformation → Caller ID for Calls from This Phone

Set:

| Field | Value |
|---|---|
| Use Device Pool Calling Party Transformation CSS | Unchecked |
| Calling Party Transformation CSS | CSS Caller ID Lab |

Save the phone configuration and apply the changes. CUCM may restart or re-register the device depending on the change and phone behavior. In this lab, the CIPC restarted and re-registered after the change.

**This originating-phone setting was the decisive change in the successful test.** The solution was not to change the ordinary Device CSS, Line CSS, PSTN route pattern, CUBE dial peer, or the shared device pool.

### 5. Leave the shared device pool and receiving phone alone

The device pool DP HQ India is shared by multiple phones. Changing its transformation CSS to fix one endpoint can affect other phones that inherit the device-pool setting.

For this single-device lab, configure the originating CIPC's own phone-level field. Do not make a global device-pool change unless the same policy is intentionally required for every affected phone.

The receiving Cisco 7821 at DN 2104 initially required no change to its ordinary call-routing CSS for the internal call to complete.

## Validation and Dialed Number Analyzer

### Functional test

1. Confirm that CIPC 2102 is registered.
2. Confirm that the Cisco 7821 at DN 2104 is registered.
3. Call 2104 from 2102.
4. Verify that the 7821 rings and note the calling identity displayed.
5. Confirm that the call remains an internal CUCM call.

Record the actual displayed number in your private lab notes. Avoid publishing the real DID unless you deliberately intend it to be public.

### DNA observation from this lab

Dialed Number Analyzer was run with originating line 2102 and dialed digits 2104. The output identified the called directory number 2104 in partition PT HQ Internal. The expanded calling-party details still showed 2102, and the relevant transformation was not obvious in the displayed analysis.

The actual phone test then succeeded after the originating CIPC's Caller ID for Calls from This Phone CSS was set to CSS Caller ID Lab.

Document these as two distinct observations:

- **Functional result:** the receiving phone presented the transformed identity during the test call.
- **Analyzer result:** the particular DNA output inspected did not make the transformation clear.

Do **not** generalize this single observation into a claim that DNA can never display phone-level transformations. Ensure that DNA is analyzing the right originating device and inspect the calling-party detail available in the exact CUCM release. For deeper verification, compare actual call behavior with the relevant signaling or CUCM traces.

## Troubleshooting: what did not work

### Mistake 1: Treating Line CSS and transformation CSS as interchangeable

The normal Line CSS and Device CSS affect destination reachability. They do not automatically apply a calling-number replacement mask.

**Lesson:** Locate the field for the specific processing stage being changed.

### Mistake 2: Mixing the caller-ID partition into the existing PSTN routing CSS

Initially, PT Caller ID Lab was added to CSS HQ PSTN Test, which already served the existing dial plan. That mixed ordinary routing policy with a transformation-only partition.

**Correction:** Create CSS Caller ID Lab and include only PT Caller ID Lab. Keep CSS HQ PSTN Test focused on its normal routing partitions.

**Lesson:** Keep route-plan policy and transformation policy separated when it improves clarity and limits unintended impact.

### Mistake 3: Changing the receiving 7821 first

The receiving phone's Remote Number section was investigated and temporarily changed during troubleshooting. That was not the setting that ultimately resolved this test.

**Correction:** Apply CSS Caller ID Lab under the originating CIPC's **Caller ID for Calls from This Phone** section.

**Lesson:** Identify whether the desired transformation should happen at the origin, on the receiving endpoint, or on the PSTN egress route. Do not change all three at once.

### Mistake 4: Enabling the External Phone Number Mask option without a reason

The lab pattern already contained an explicit Calling Party Transformation Mask.

**Correction:** Leave Use Calling Party's External Phone Number Mask off for the explicit-mask test.

**Lesson:** Change one transformation method at a time and verify it.

### Mistake 5: Assuming a successful internal call must pass through CUBE

The call from 2102 to 2104 was an internal CUCM call. A successful internal call does not prove anything about CUBE-to-Twilio or CUBE-to-LiveHub transport.

**Lesson:** Keep internal call routing, PSTN egress, and external AI connectivity as separate test paths.

## Scaling to more phones

The working configuration uses an exact pattern, 2102. It does **not** automatically match 2103, 1003, or future phones.

First decide which of these policies is actually wanted:

### Option A — Every 21xx extension shares the same caller identity

A broader pattern such as **21XX** can be considered to match a 21xx range, subject to validation against the CUCM pattern syntax and numbering plan in use. A fixed transformation mask means every matched calling number becomes the same configured DID.

For extension 1003, either create a separate exact transformation pattern for 1003 or choose a broader pattern only if applying it to every number in that range is intentional.

Before expanding the pattern:

1. Preserve the working 2102 configuration as a baseline.
2. Check that the broader pattern cannot overlap or conflict with other transformation patterns.
3. Apply the dedicated transformation CSS to each intended originating phone, or use a controlled device-template/Bulk Administration workflow.
4. Test more than one source extension, including a number that should not match.
5. Confirm internal display and PSTN caller-ID behavior separately.

### Option B — Each user has a distinct E.164 identity

Do not use one fixed mask for all callers. Build a documented extension-to-DID mapping and use the organization's approved calling-party transformation design, such as per-DN external phone number masks or explicit transformations.

A single Twilio DID is a shared identity, not a unique DID for every user.

### Enterprise rollout guidance

- Pilot with one source phone first.
- Record the before/after device settings.
- Use the dedicated transformation partition and CSS consistently.
- Roll out only to the intended group of phones.
- Validate internal calls and outbound PSTN calls independently.
- Maintain a mapping and clear ownership for every presented DID.
- Avoid a shared device-pool change unless all phones using that pool should inherit the same policy.

## Change control and rollback

### Before making changes

- [ ] Record the original originating-phone transformation settings.
- [ ] Record the existing CSS membership and transformation pattern.
- [ ] Confirm authorization to present the selected caller ID.
- [ ] Ensure that the dedicated partition/CSS is not being reused unexpectedly.
- [ ] Define expected behavior for internal calls and PSTN calls separately.

### Rollback

1. Restore the originating CIPC's original Caller ID for Calls from This Phone settings.
2. Restore prior CSS membership and transformation-pattern configuration from the recorded baseline.
3. Allow the phone to re-register if needed.
4. Verify that the internal 2102-to-2104 call works and returns to the original displayed caller identity.

Do not delete a partition or CSS until you verify that no other patterns or phones reference it.

## References

Use documentation for the exact CUCM 14 release and SU running in the lab.

- [Cisco Unified Communications Manager Administration Guide](https://www.cisco.com/c/en/us/support/unified-communications/unified-communications-manager-callmanager/products-maintenance-guides-list.html)
- [Cisco Unified Communications Manager Dialed Number Analyzer documentation](https://www.cisco.com/c/en/us/support/unified-communications/unified-communications-manager-callmanager/products-user-guide-list.html)
- [Cisco Unified CM transformation configuration documentation](https://www.cisco.com/c/en/us/support/unified-communications/unified-communications-manager-callmanager/products-configuration-examples-list.html)

Field names and menus can vary by release and SU. Confirm them in the administration interface and documentation matching the deployed version.

## Disclaimer

This repository records a lab troubleshooting result. It is not an official Cisco design guide. Test any adaptation in a controlled environment before using it in production.

No license is specified in this repository.
