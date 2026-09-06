# Project Guide

> **Audience:** clients, managers, operators, drivers, and everyone who is not writing code.\
> This guide explains what the project is, where it stands today, and how we all work together. Technical readers should see the [Technical Documentation](../non-technical/TECHNICAL.md).

***

## 1. Hello and Welcome

This booklet is your non-technical companion to the Navon project — a vehicle tracking system. You do not need to understand programming, databases, or networking to follow it. It tells you:

* **What we are building** and why it matters,
* **Where the project stands today** (what is real, what is still open),
* **What has been done** so far,
* **What is still to be done**,
* **How to use the system day-to-day** (step-by-step guides),
* **How to get involved** and how the team works together — including a short _"How can I contribute?"_ questionnaire and the questions we still need to answer together to move the project forward.

***

## 2. What Is Navon?

**The project is now called Navon.** (The name is a first decision. The domain `navon.com` is not available, so the web address is still open — and the brand — logo, colors, fonts — is wide open too; that is something we decide together.)

Navon is a **fleet management and vehicle-tracking platform**. It helps organizations know **where their vehicles are, right now**, and what those vehicles are doing.

Imagine you manage a fleet of delivery vans, service cars, or construction trucks. With Navon you can:

* See **every vehicle on a live map** — where it is, how fast it is going, and whether the engine is on.
* **Draw safe zones** around places like your yard or a client site, and be **alerted when a vehicle enters or leaves** a zone.
* Get **warnings** for unsafe or unusual behavior — speeding, harsh braking, harsh cornering, a weak battery, or a vehicle going offline.
* Look back at **where a vehicle has been** (history), what **trips** it made, and its **fuel/battery and power health** over time.
* Manage everything from a **web dashboard**, an **Android mobile app** (with fingerprint login), and — for testing — a **GPS simulator** that acts like a real tracker.

The system is designed from the ground up for **multiple companies (tenants)**. Each customer or organization has its own private area — its own vehicles, drivers, settings, and alerts — while a system administrator can see and manage everyone.

### Why it matters

* **Safety** — spot dangerous driving before it causes an accident.
* **Security** — know when a vehicle leaves an approved area.
* **Savings** — reduce fuel waste, idling, and unauthorized use.
* **Trust** — share accurate, real-time information with customers and management.
* **Growth** — one platform, many companies, no per-client installations.

***

## 3. Where the Project Stands Today

The system works — this is a **working foundation, not yet a finished product or a finished business**.

* **The software is built and running.** A good amount of time has been spent building the actual working system. The goal at this stage was not to make it look like a finished commercial product, but to prove that the idea works and to build the important features behind it. See [What Has Been Done](non-technical.md#5-what-has-been-done) and [What Still Needs to Be Done](non-technical.md#6-what-still-needs-to-be-done).
* **Hosting is temporary.** The system is currently hosted on a server at **akako.net:8082**. This is a development and testing setup (the server also runs other projects its owner is involved in), so it should not be considered the final hosting. There is no dedicated domain yet.
* **The Android app works but is not published.** It is mainly used for testing and development. Final app configuration, testing, publishing, and the other Play Store requirements still need to be done.
* **UI/UX and product design are still to come.** The current interface was made to be usable and to test functionality. It is not the final design. We still need to improve the look and feel, make the system easier to use, settle the overall design style, and make the web and mobile applications feel like one complete product.
* **Branding is not finalized.** We need to decide the logo, colors, fonts, and general identity of the platform — the name is now **Navon**. This is something we can all work on together, because the product should have its own identity before we present it publicly.
* **The business side is open.** There is no finalized business plan, pricing model, target-customer definition, sales strategy, or clear decision on how the service will be offered. The technology is one part of the project; turning it into a real business requires working on these areas too.
* **More features are planned and some need refinement.** Fuel estimation, driver performance scoring, route optimization, maintenance reminders, better notifications, analytics, data export, and other improvements. Some existing features also need more real-world testing.

**Why we are sharing this now:** not to say everything is ready, but so everyone can see what has already been done and we can discuss what to build from here.

***

## 4. Who Uses It

| Who                          | What they do                                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **System Administrator**     | Owns the whole platform. Creates companies, assigns their admins, watches all devices, and can send commands to trackers. |
| **Company Admin**            | Runs one company's fleet day-to-day. Adds users and vehicles, links trackers, draws safe zones, and tunes settings.       |
| **Company User**             | Views the company's vehicles on the map and dashboard (read-only).                                                        |
| **Driver**                   | Sees only their own assigned vehicle(s) (read-only).                                                                      |
| **Fleet/Operations Manager** | Watches the dashboard, live map, and alerts to keep the fleet running smoothly.                                           |

***

## 5. What Has Been Done

The core platform is built and working. Here is the plain-language scoreboard.

### Done and in use

* ✅ **Accounts & login** — people register with email/password; the mobile app can use fingerprint unlock.
* ✅ **Multi-company structure** — each organization gets its own isolated fleet and settings.
* ✅ **Live map** — vehicles appear in real time with speed, engine state, and power status (green = external power, yellow = battery, gray = offline).
* ✅ **Trackers connect automatically** — a new device only needs to be powered on and set server address; it registers itself. No technical setup required.
* ✅ **Vehicles & drivers** — link a tracker to a vehicle and assign vehicles to drivers. One device, one vehicle.
* ✅ **Safe zones (geofences)** — draw a circle or a polygon on the map, set a speed limit for the zone, and get alerts for enter/exit and speeding.
* ✅ **Alerts & notifications** — warnings for speeding, hard braking, hard cornering, low battery, going offline, and geofence speeding, with a small delay guard so you are not flooded.
* ✅ **Driving behavior events** — the system logs harsh acceleration, hard braking, cornering, idling, driving with the engine off, and signal jamming.
* ✅ **Power monitoring** — track external/battery voltage and know which vehicles are running on backup power.
* ✅ **History & trips** — replay where a vehicle went and see its trips (engine-on segments).
* ✅ **Remote control of devices** — send commands to a tracker (e.g., reset or config) from the dashboard.
* ✅ **Android mobile app** — the dashboard in your pocket, secured with biometrics.
* ✅ **GPS simulator** — a training/testing tool that mimics one or many trackers driving smooth, realistic routes — ideal for demos, training, and testing without hardware.
* ✅ **Central logging** — all system activity is logged in one place (Seq) so problems are easier to find and fix, and it works for development too.

> Not everything is 100% verified: remote commands to physical devices, harsh-acceleration detection,warning for speeding notification have been implemented but not fully validated with real hardware on the road. That is part of the **to-do** below.

***

## 6. What Still Needs to Be Done

These will be the upcoming agreed-upon next steps. They help the team know what to work on and what to expect.

### Finish and validate (short term)

* ⏳ **Validate remote commands** (CPU reset, settings via the mobile/data connection) with real trackers on the road.
* ⏳ **Fine-tune driving alerts** (hard acceleration/braking/curve) using real driving data.
* ⏳ **Round out reports and exports** so management can pull data easily.

### Planned improvements (medium term)

* Fuel-consumption estimation
* Driver behavior scoring and ranking (using smart analysis)
* Route optimization
* Scheduled maintenance alerts (service reminders)
* Multi-language support (Afan Oromo, Amharic, …)
* iOS mobile app
* Push notifications to phones (even when the app is closed)
* Data export (CSV/Excel/PDF)
* Live dashboard analytics charts and geofence heatmaps
* Two-factor authentication and audit trails for admins
* Integration with more devices

### Bigger ideas (long term)

* OBD-II engine data extensions
* Customer/CRM management built into the platform
* Full Fleet management system
* Integration with ERP systems (Erpnext, Odoo...)

> **How to add to this list:** the Product Owner collects and prioritizes requests. See [Working Together](non-technical.md#8-working-together) for how to raise an idea.

***

## 7. How-To Guides (Day-to-Day)

### 7.1 Get an account

1. Open the app or website and click **Create One**. (http://akako.net:8082/)
2. Enter your email, a password, and your name.
3. You are logged in automatically. Until a Company Admin links you to a company, you will see an empty map.

### 7.2 First-time setup (System Administrator)

1. Sign in with the administrator account that is created automatically on first start. (Username: admin@gpstracker.local, Password: P@ssw0rd)
2. **Create a company** for each customer/organization: `Administration → Companies → Create`.
3. **Tune company rules** — how many minutes before a vehicle is considered "offline", and how much of a pause splits one trip from the next.
4. **Assign a Company Admin** to each company (users must register first — see 7.1).
5. **Assign the Company Admin** next to the company: `Administration → Companies → Users → Add`.

### 7.3 Add users and set what they can see

1. Company Admin: go to `Admin → Link existing`, enter the person's registration email, and link them.
2. Choose their **role**:
   * **Admin** — manages users, vehicles, trackers, and zones for the company;
   * **User** — can view all company vehicles;
   * **Driver** — sees only the vehicles assigned to them.
3. For **Drivers**, tick the vehicles they should see (`Vehicles` checkbox).

### 7.4 Put a tracker in a vehicle

1. **Install** the tracking device in the vehicle and power it on.
2. The platform **detects it automatically** — you will see _"New device registered: (IMEI)"_.
3. Give it a friendly name (e.g., "Truck 01"): `Devices → {device} → Rename`.
4. **Assign it to your company**: device detail → `Companies → Edit`.
5. Company Admin: create the **vehicle** (`Vehicles → Create vehicle`), then **link the device** to it. One device links to only one vehicle at a time.

> No manual device registration is required — power on and it appears.

### 7.5 Draw a safe zone (geofence)

1. Go to `Geofences → Create zone`.
2. Choose a **circle** (center + radius) or **polygon** (draw the outline).
3. Give it a name and (optional) a **speed limit** for that area.
4. **Assign vehicles** to the zone.
5. You are now notified when an assigned vehicle **enters or exits** the zone, and if it **exceeds the zone's speed limit**.

### 7.6 Watch the fleet (map & dashboard)

* **Live Map** — see all vehicles, click one for details (speed, engine, power, telemetry such as doors, GSM signal, odometer).
* **Dashboard** — a summary card: total vehicles, online now, offline count, and the latest zone events.
* **Device page** — for any vehicle: history replay map, list of trips, power health, events, and zone violations.

### 7.7 Read and act on alerts

* Alerts appear as **notifications** with a badge count (bell icon).
* Click to read; mark as **read** or dismiss.
* Settings (Company Admin) control **which** alerts are on and **the thresholds** (e.g., speed limit, battery voltage, force for braking).

### 7.8 Change alert & company settings (Company Admin)

* `Settings` page: enable/disable each alert type, set values, set the company **offline threshold**, and set the **trip gap**.

***

## 8. Working Together

A project succeeds when everyone — technical and non-technical — knows their part. Here is a tiypical software team runs.

### 8.1 Who is on the team

| Role                        | What they own                                                           |
| --------------------------- | ----------------------------------------------------------------------- |
| **Business Owner / Client** | The goal and budget. Decides what success looks like.                   |
| **Product Owner**           | Decides **what** gets built and in what order (the priority list).      |
| **Project Manager**         | Coordinates people, timing, and communication; keeps everyone informed. |
| **Developers**              | Build and maintain the software.                                        |
| **QA / Testers**            | Check that things work and catch problems before users do.              |
| **UIUX**                    | Make sure it looks and feels right.                                     |
| **Clients User**            | Use the system daily and report what works — or does not.               |

### 8.2 How the team moves together

* **Plan together first.** Work is picked in small, reviewable batches from the priority list — not all at once.
* **Short cycles.** The team works in short sprints (1–2 weeks), then shows results. Nothing big is delivered in one giant, unannounced release.
* **Show and tell.** At each review, the Product Owner and stakeholders see what was built. Feedback is captured and goes back into the priority list.
* **Small, reviewable changes.** The team prefers many small updates over a few large surprises — easier to test, easier to fix, easier to follow.
* **Decisions and changes are tracked.** Big choices (new features, changes in scope, new costs) are agreed and recorded, so there are no surprises later.

### 8.3 An open invitation — participate freely

We would really like everyone to participate — but **only if they are genuinely interested in the idea and willing to contribute**. Not everyone needs to work on the technical side; there are many different areas to help with.

* Some people can help by **testing the existing system** and reporting problems or suggesting improvements.
* Others can look at the system from a **normal user's perspective** and tell us what is confusing or what could be easier.
* We can also work together on **UI/UX, branding, product ideas, business planning, pricing, marketing, customer requirements, and future features**.
* If you have experience or interest in **transportation, logistics, fleet management, business, design, sales**, or anything related, your experience can be very useful.

> **Please decide freely.** The main goal at this point is to bring together people who are genuinely interested in the project, show what has already been done, get honest feedback, and decide what we should focus on next. I also want everyone to feel free to participate only if they are truly interested in the idea and willing to contribute. There should be no pressure to join because of possible future profit, expectations from others, or simply because friends are involved.I really want to stress this point because we may have some people in our circle who are a bit "officious" and tend to push their opinions or ideas onto others. At the same time, we may also have people who are push-overs who are more likely to simply follow whatever they are told. I don't want either of these situations to influence anyone's decision to participate. Everyone should feel free to make their own decision based on their genuine interest in the project, not because of pressure from others, friendship, or the expectation of future profit..

### 8.4 How you can participate (non-technical people)

You do not need to read code to make a real difference:

* **Test and report.** Use the live map, zones, and alerts. When something looks wrong, tell the team exactly what you did and what you saw (screenshots help a lot).
* **Give real-world feedback.** Tell the Product Owner what feels natural and what feels awkward. You are the expert on your fleet.
* **Define rules and thresholds.** The system only works well with real numbers — your speed limits, your battery expectations, the color and the layout, the flow of a process, your "offline" timeouts. These are business decisions, not technical ones.
* **Help shape the product and the brand.** UI/UX, branding (name is Navon — logo, colors, and fonts are still open), product ideas, business planning, pricing, marketing, and customer requirements all need real input.
* **Request improvements.** Every idea goes on the priority list. Not everything is built immediately, but everything is heard and ranked.
* **Supply what only you can:** the list of vehicles, drivers, zones, the tracking hardware, and the SIM cards/data.
* **Validate the results.** After a release, confirm that the behavior you asked for actually shows up in practice.

### 8.5 Moving forward together

Going forward, we also want to change how the project is presented. In earlier notes, it was written as "I" because one person described the work they personally did. If we move forward as a group, the project should no longer be presented as something that belongs to or is driven by one person. We should find a suitable way of reporting as **"we"**, and ideally have someone responsible for coordinating and presenting project updates on behalf of the group. This makes the project feel like a shared effort and helps us establish clear roles and responsibilities as we move on.

### 8.6 Ground rules

* One priority list, maintained by the Product Owner — so effort focuses on what matters most.
* Nothing is called "done" until a person who will actually use it has confirmed it works.
* Honest status over perfect news. If something slips, the team says so early and explains the plan.
* Questions are always welcome — there are no silly questions in this project. If you are unsure, ask.

***

## 9. How Can I Contribute?

This is the most useful place for a new person to start. Whether you bring **time, expertise, funding, contacts, or ideas**, there is a way to help. If you are considering joining, answer these six questions for yourself and bring them to a conversation:

What interests you about the project? e.g. I want to help with the business side. What could you contribute? e.g Time, expertise, funding, contacts, or ideas. What role would you like to explore? e.g. Co-founder, advisor, volunteer, partner, or supporter. What would you like to help accomplish? e.g Organize the team, develop the brand, or find funding. What do you need from the project? e.g More information, a clear role, or a discussion. What should we do next? e.g Meet, propose an idea, or take responsibility for a task.

### Ways you can help

* **Time** — organizing, testing, writing, communicating, or managing.
* **Expertise** — business planning, finance, marketing, design, or technology.
* **Funding** — one-off support, a monthly contribution, or introducing sponsors.
* **Contacts** — opening doors to clients, partners, or organizations.
* **Ideas** — direction on how the product, brand, or team should evolve.

> There is no "right" way to contribute. A useful idea, an honest opinion, or an introduction is just as valuable as money or code.

***

## 10. Questions to Move the Project Forward

The project is beyond the "what is it" stage, but a number of decisions still need to be made together. Use these as a discussion agenda — pick a theme per meeting, or split the questions among the team and compare answers.

### 10.1 Brand & identity

* What should the project be called? — **decided: Navon** (note: `navon.com` is unavailable, so the web address is still open)
* Who is the brand for?
* What should be the brand color and theme?
* What should people understand when they first hear about it?
* What values should the brand represent?
* What should the visual identity look and feel like?
* What should make the project recognizable?
* Who should help decide the name and branding?
* What should we prepare before presenting the project publicly?

### 10.2 Vision, funding & costs

* What would success look like in 1 year?
* What values should guide the project?
* What will it cost to move the project forward?
* What costs do we already know about?
* What costs are we likely to discover later?
* What can we do with the resources we already have?
* Who is willing to contribute financially?
* Should contributions be donations, investments, membership fees, or something else?
* Are there grants, sponsors, or partnerships we should explore?
* What would we need funding for?
* How much funding would allow us to reach the next major milestone?
* How should financial contributions be recorded and managed?
* What should happen if the project does not generate revenue?

### 10.3 People, support & advice

* Who is already interested?
* Who else should we invite?
* What skills, experience, or connections are missing?
* What skills, experience, or connections do we need?
* What kinds of people would add the most value?
* What would make someone want to join?
* What would make someone stay involved?
* Who could advise us?
* What expertise do we need from outside the team?
* Who could introduce us to useful people or organizations?
* What should we ask experienced people to review?
* What assumptions should we ask others to challenge?
* What lessons can we learn from similar initiatives?
* What partnerships could help us?
* What kind of feedback would be most valuable right now?

### 10.4 Ways to contribute & collaborate

* What are the different ways someone can contribute?
* Can people contribute time, money, expertise, contacts, or ideas?
* What responsibilities could be shared?
* How should people propose ideas?
* How should we decide which ideas to pursue?
* How should we recognize people's contributions?
* How can we make collaboration easy for someone joining for the first time?

### 10.5 Team, decisions & organization

* What kind of team do we need at this stage?
* What roles are necessary?
* Who is currently responsible for what?
* Who should coordinate the project?
* Who should make major decisions?
* How should responsibilities be divided?
* How should we handle disagreements?
* How should we communicate?
* What should happen if someone becomes unavailable, lazy, untrustworthy, theif? (yeselel?)
* Does this project need a formal organization now?
* What would be the benefits of creating a company?
* What posibilities are there to work without forming any organization, demboch tila sayhonuben?
* What would each option allow us to do?
* What responsibilities would come with each option?
* How should ownership or membership be handled?
* What should happen if new people join later?
* What should happen if someone leaves?
* What agreements should we have before making the project formal?

### 10.6 Next steps

* What is the most important thing we need to accomplish next?
* What should we discuss at the next meeting?
* What should we decide before inviting more people?
* What should we prepare before presenting the project to potential collaborators?
* Who should take responsibility for the next steps?
* What can each interested person contribute in the next few weeks?
* What would make us ready to move from an informal project to a more organized initiative?
* What should we review after the next milestone?

### What should happen next?

1. **Choose a coordinator** — one person owns the next step so it does not get lost.
2. **Pick one theme** from this list and make it the agenda of the next meeting.
3. **Agree on one decision** to make before inviting more people in.
4. **Ask everyone for one contribution** they can make in the next few weeks.
5. **Bring potential collaborators in early** — with a short, honest summary of the project, what is done, and what we are deciding.

***

## 11. What and Who Has Contributed So Far

Notable contributions have already been made — before any formal budget or investment is decided:

* **Wube (business & fleet-management side)** contributes time and practical knowledge: how the different GPS devices work, how they are configured and set up, testing devices in different situations, and providing personal SIM cards as well as additional SIM cards for testing. He has also provided access and credentials for several commercial systems and services (paid and free) used for testing, comparison, and understanding how similar systems work.
* **Technical side (the developer)** builds the system, puts the parts together, deploys it, tests it, fixes issues, and continues development — supported along the way by various tools and AI systems for development, research, troubleshooting, and speeding up the work.
* **Other individuals** have helped here and there, but in small amounts, so it is not listed separately. That is not because people did not want to help — at this early stage, not much extra help was needed.

There is no formal financial investment yet. Both main sides have already contributed time, knowledge, resources, devices, SIM cards, services, and technical work to get the project to its current stage.

***

## 12. Cost & Budget

> These are initial estimates for the next stage of the project. The actual cost can change depending on the options we choose, the people involved, and whether some work can be done by members of the group.

### 12.1 Initial project budget (next stage)

| Area                        | What is Needed                                                                      | Estimated Cost  |
| --------------------------- | ----------------------------------------------------------------------------------- | --------------- |
| Dedicated VPS / Hosting     | Separate server for the project, database, backups, and production environment      | $300–600 / year |
| Domain & SSL                | Domain name and related services                                                    | $20–50 / year   |
| UI/UX Design                | Improve the web and mobile interface and make the product easier to use             | TBD             |
| Branding                    | Logo, colors, fonts, basic brand identity, and marketing materials                  | TBD             |
| Android App                 | Final testing, preparation, Play Store setup and publishing                         | TBD             |
| iOS App                     | Initial development and Apple Developer setup                                       | TBD             |
| Testing & Devices           | GPS trackers, SIM cards, testing devices, and other hardware for real-world testing | TBD             |
| Software Development        | Completing unfinished features, fixing issues, improving existing functionality     | TBD             |
| Security & Production Setup | Backups, monitoring, security improvements, domain/email configuration              | TBD             |
| Business & Legal            | Business registration, agreements, basic legal/accounting requirements              | TBD             |
| Marketing & Website         | Public website, product presentation, initial marketing materials                   | TBD             |
| **Estimated initial total** |                                                                                     | **TBD**         |

### 12.2 An important note on the budget

* These are **rough estimates, not a final investment requirement**. The actual amount can be significantly lower if members of the group contribute their own skills — someone handles UI/UX, another works on branding, another helps with business planning, and technical members continue development.
* **Do not start by spending a large amount of money.** First agree on the direction, test the existing system, and identify what is actually needed — then spend where it provides the most value.
* **Keep costs separate:** development costs, business expenses, and future operational costs should be tracked separately. This makes it easier to understand what has already been contributed, what still needs to be invested, and how future expenses or income should be handled.

### 12.3 Budget by stages

| Stage                   | Main Goal                                                              | Approx. Budget |
| ----------------------- | ---------------------------------------------------------------------- | -------------- |
| Stage 1                 | Testing, feedback, branding and basic product design                   | TBD            |
| Stage 2                 | Production hosting, domain, security and completing important features | TBD            |
| Stage 3                 | Mobile app publishing, real-world device testing and website           | TBD            |
| Stage 4                 | Business setup, marketing and initial operations                       | TBD            |
| **Total initial range** |                                                                        | **TBD**        |

### 12.4 Hardware per device (indicative market prices)

| Item                            | Unit Cost   | Quantity | Total                              |
| ------------------------------- | ----------- | -------- | ---------------------------------- |
| Teltonika FMB120 tracker        | $50–80      | 10       | $500–800                           |
| Teltonika FMB920 tracker        | $40–60      | 10       | $400–600                           |
| SIM card (data)                 | $5–10/month | 20       | $100–200/month                     |
| OBD-II cable                    | $10–20      | 20       | $200–400                           |
| **Total hardware (20 devices)** |             |          | **$1,100–1,800 + $100–200/mo SIM** |

### 12.5 How we compare with commercial platforms

| Solution                 | Setup | Monthly (50 vehicles) |
| ------------------------ | ----- | --------------------- |
| **Navon (this project)** | TBD   | TBD                   |
| Teltonika Flespi         | $0    | $150–300              |
| GPS-Server.net           | $0    | $200–400              |
| Wialon                   | $0    | $300–600              |
| Gurtam                   | $0    | $400–800              |

Because we control the platform ourselves, running costs are mainly the server and hosting — there are no per-vehicle licensing fees.

***

## 13. Where to Go for More

* [**Technical Documentation**](../non-technical/TECHNICAL.md) — architecture, APIs, database, and deployment, for the technical team.
* [**Web Dashboard README**](../non-technical/web/) — notes for the frontend developers.

_Questions about the project? Ask the Product Owner or Project Manager — they will point you to the right place._
