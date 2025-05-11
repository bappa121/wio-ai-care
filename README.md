import uuid
import datetime
from collections import defaultdict

# --- Data Structures (Conceptual) ---
# We'll use dictionaries for simplicity. In a real app, these would be database models.

class Citizen:
    def __init__(self, umn, name, dob_str, contact_info="N/A"):
        self.umn = umn
        self.name = name
        try:
            self.dob = datetime.datetime.strptime(dob_str, "%Y-%m-%d").date()
        except ValueError:
            raise ValueError("DOB must be in YYYY-MM-DD format")
        self.contact_info = contact_info
        self.medical_records = []  # List of dictionaries
        self.prescriptions = []    # List of dictionaries (active prescriptions)

    def __str__(self):
        return f"UMN: {self.umn}, Name: {self.name}, DOB: {self.dob}"

class WioAICareSystem:
    def __init__(self):
        self.citizens_data = {} # UMN -> Citizen object
        self.medicine_reminders = defaultdict(list) # UMN -> list of reminder tasks

    def generate_umn(self):
        """Generates a unique medical number."""
        return str(uuid.uuid4())

    def register_citizen(self, name, dob_str, contact_info="N/A"):
        """Registers a new citizen and assigns a UMN."""
        umn = self.generate_umn()
        if umn in self.citizens_data:
            # Extremely unlikely with UUID, but good practice
            return self.register_citizen(name, dob_str, contact_info)
        
        try:
            citizen = Citizen(umn, name, dob_str, contact_info)
            self.citizens_data[umn] = citizen
            print(f"Citizen {name} registered with UMN: {umn}")
            return umn
        except ValueError as e:
            print(f"Error registering citizen: {e}")
            return None


    def get_citizen(self, umn):
        """Retrieves citizen data by UMN."""
        return self.citizens_data.get(umn)

    def add_medical_record(self, umn, date_str, doctor, diagnosis, symptoms, treatment_notes, attachments=None):
        """Adds a medical record for a citizen."""
        citizen = self.get_citizen(umn)
        if not citizen:
            print(f"Error: Citizen with UMN {umn} not found.")
            return False

        try:
            record_date = datetime.datetime.strptime(date_str, "%Y-%m-%d").date()
        except ValueError:
            print("Error: Record date must be in YYYY-MM-DD format.")
            return False

        record = {
            "id": str(uuid.uuid4()), # Unique ID for the record
            "date": record_date,
            "doctor": doctor,
            "diagnosis": diagnosis,
            "symptoms": symptoms if isinstance(symptoms, list) else [symptoms],
            "treatment_notes": treatment_notes,
            "attachments": attachments or [] # e.g., paths to lab reports
        }
        citizen.medical_records.append(record)
        citizen.medical_records.sort(key=lambda r: r["date"], reverse=True) # Keep sorted
        print(f"Medical record added for UMN {umn} on {date_str}.")
        return True

    def add_prescription(self, umn, medicine_name, dosage, frequency_per_day, start_date_str, duration_days):

        """Adds a medicine prescription for a citizen."""
        citizen = self.get_citizen(umn)
        if not citizen:
            print(f"Error: Citizen with UMN {umn} not found.")
            return False

        try:
            start_date = datetime.datetime.strptime(start_date_str, "%Y-%m-%d").date()
        except ValueError:
            print("Error: Prescription start date must be in YYYY-MM-DD format.")
            return False
        
        end_date = start_date + datetime.timedelta(days=duration_days)

        prescription = {
            "id": str(uuid.uuid4()),
            "medicine_name": medicine_name,
            "dosage": dosage, # e.g., "1 tablet", "10mg"
            "frequency_per_day": frequency_per_day, # e.g., 1, 2, 3
            "start_date": start_date,
            "duration_days": duration_days,
            "end_date": end_date,
            "is_active": True # Will be set to False later if needed
        }
        citizen.prescriptions.append(prescription)
        # Sort by end_date to easily find active ones
        citizen.prescriptions.sort(key=lambda p: p["end_date"], reverse=True) 
        print(f"Prescription for {medicine_name} added for UMN {umn}.")
        self._schedule_reminders(umn, prescription) # Schedule initial reminders
        return True

    def _update_active_prescriptions(self, umn):
        """Internal helper to mark past prescriptions as inactive."""
        citizen = self.get_citizen(umn)
        if not citizen: return
        today = datetime.date.today()
        for p in citizen.prescriptions:
            if p["end_date"] < today:
                p["is_active"] = False

    def get_active_prescriptions(self, umn):
        """Gets currently active prescriptions for a citizen."""
        citizen = self.get_citizen(umn)
        if not citizen:
            print(f"Error: Citizen with UMN {umn} not found.")
            return []
        self._update_active_prescriptions(umn)
        return [p for p in citizen.prescriptions if p["is_active"]]

    def _schedule_reminders(self, umn, prescription):
        """(Simplified) Schedules reminders for a prescription."""
        # In a real system, this would interact with a task scheduler (Celery, cron, etc.)
        # or a push notification service.
        # For this demo, we'll just add to an in-memory list.
        
        # Clear old reminders for this UMN for this specific medicine to avoid duplicates if re-added
        self.medicine_reminders[umn] = [
            r for r in self.medicine_reminders[umn] if r.get("prescription_id") != prescription["id"]
        ]

        if not prescription["is_active"]:
            return

        # Simplified reminder logic: assumes doses are evenly spaced throughout the day.
        # e.g., 3 times a day: 8 AM, 2 PM, 8 PM
        reminder_times_of_day = {
            1: [datetime.time(9, 0)],  # 9 AM
            2: [datetime.time(8, 0), datetime.time(20, 0)], # 8 AM, 8 PM
            3: [datetime.time(8, 0), datetime.time(14, 0), datetime.time(20, 0)], # 8 AM, 2 PM, 8 PM
            4: [datetime.time(7,0), datetime.time(12,0), datetime.time(17,0), datetime.time(22,0)]
        }
        
        times_to_take = reminder_times_of_day.get(prescription["frequency_per_day"], [datetime.time(9,0)]) # Default once a day

        for i in range(prescription["duration_days"]):
            current_reminder_date = prescription["start_date"] + datetime.timedelta(days=i)
            if current_reminder_date < datetime.date.today(): # Don't schedule for past days
                continue

            for time_of_day in times_to_take:
                reminder_datetime = datetime.datetime.combine(current_reminder_date, time_of_day)
                self.medicine_reminders[umn].append({
                    "prescription_id": prescription["id"],
                    "medicine_name": prescription["medicine_name"],
                    "dosage": prescription["dosage"],
                    "reminder_time": reminder_datetime,
                    "acknowledged": False
                })
        
        # Sort reminders by time
        self.medicine_reminders[umn].sort(key=lambda r: r["reminder_time"])
        # print(f"Scheduled {len(times_to_take) * prescription['duration_days']} reminders for {prescription['medicine_name']} for UMN {umn}")


    def get_upcoming_reminders(self, umn, lookahead_hours=24):
        """Gets upcoming medicine reminders for a citizen."""
        citizen = self.get_citizen(umn)
        if not citizen:
            print(f"Error: Citizen with UMN {umn} not found.")
            return []
        
        self._update_active_prescriptions(umn) # Ensure prescription active status is current
        
        now = datetime.datetime.now()
        upcoming = []
        active_prescription_ids = {p["id"] for p in self.get_active_prescriptions(umn)}

        # Filter reminders: only for active prescriptions and not past
        valid_reminders = []
        for reminder in self.medicine_reminders.get(umn, []):
            if reminder["prescription_id"] in active_prescription_ids and \
               reminder["reminder_time"] > now and \
               not reminder["acknowledged"]:
                valid_reminders.append(reminder)
        
        self.medicine_reminders[umn] = valid_reminders # Clean up old/acknowledged ones from main list

        # Now select from valid_reminders based on lookahead
        lookahead_limit = now + datetime.timedelta(hours=lookahead_hours)
        for reminder in self.medicine_reminders.get(umn, []): # Use the cleaned list
            if now < reminder["reminder_time"] <= lookahead_limit and not reminder["acknowledged"]:
                 upcoming.append(reminder)
        
        return sorted(upcoming, key=lambda r: r["reminder_time"])
    
    def acknowledge_reminder(self, umn, reminder_time_str):
        """Marks a specific reminder as acknowledged (taken)."""
        try:
            reminder_time_to_ack = datetime.datetime.strptime(reminder_time_str, "%Y-%m-%d %H:%M:%S")
        except ValueError:
            print("Invalid datetime format for acknowledgment. Use YYYY-MM-DD HH:MM:SS")
            return False

        found = False
        for reminder in self.medicine_reminders.get(umn, []):
            if reminder["reminder_time"] == reminder_time_to_ack and not reminder["acknowledged"]:
                reminder["acknowledged"] = True
                print(f"Reminder for {reminder['medicine_name']} at {reminder_time_to_ack} acknowledged.")
                found = True
                break
        if not found:
            print(f"No pending reminder found for UMN {umn} at {reminder_time_to_ack}.")
        return found


    def ai_analyze_medical_data(self, umn):
        """(Simulated) AI analysis of medical data."""
        citizen = self.get_citizen(umn)
        if not citizen:
            return "Error: Citizen not found."

        insights = []
        if not citizen.medical_records:
            return "No medical records to analyze."

        # 1. Check for frequent symptoms
        symptom_counts = defaultdict(int)
        for record in citizen.medical_records:
            for symptom in record["symptoms"]:
                symptom_counts[symptom.lower()] += 1
        
        for symptom, count in symptom_counts.items():
            if count >= 3: # Arbitrary threshold
                insights.append(f"Potential Recurring Symptom: '{symptom}' reported {count} times.")

        # 2. Flag potentially chronic conditions based on keywords
        chronic_keywords = ["diabetes", "hypertension", "asthma", "arthritis", "chronic"]
        diagnoses_texts = " ".join([record["diagnosis"].lower() for record in citizen.medical_records])
        for keyword in chronic_keywords:
            if keyword in diagnoses_texts:
                insights.append(f"Potential Chronic Condition Indicated: System detected keyword '{keyword}'.")
        
        # 3. Check for multiple medications (potential polypharmacy)
        active_prescriptions = self.get_active_prescriptions(umn)
        if len(active_prescriptions) >= 5: # Arbitrary threshold
            insights.append(f"Polypharmacy Alert: Citizen is on {len(active_prescriptions)} active medications. Review for interactions.")

        # 4. Check for recent serious diagnosis
        serious_keywords = ["cancer", "heart attack", "stroke", "malignant"]
        if citizen.medical_records:
            latest_record = citizen.medical_records[0] # Assumes sorted by date
            for keyword in serious_keywords:
                if keyword in latest_record["diagnosis"].lower():
                    insights.append(f"Alert: Recent serious diagnosis detected - '{latest_record['diagnosis']}' on {latest_record['date']}.")
        
        return insights if insights else ["No specific AI insights found based on current rules."]

    def summarize_citizen_medical_data(self, umn):
        """Generates a summary of a citizen's medical data."""
        citizen = self.get_citizen(umn)
        if not citizen:
            return "Error: Citizen not found."

        self._update_active_prescriptions(umn) # Ensure status is current

        summary = f"--- Medical Summary for {citizen.name} (UMN: {umn}) ---\n"
        summary += f"Date of Birth: {citizen.dob}\n"
        summary += f"Contact: {citizen.contact_info}\n\n"

        summary += f"Total Medical Records: {len(citizen.medical_records)}\n"
        if citizen.medical_records:
            latest_record = citizen.medical_records[0] # Assumes sorted
            summary += f"Last Visit: {latest_record['date']} with Dr. {latest_record['doctor']}\n"
            summary += f"  Latest Diagnosis: {latest_record['diagnosis']}\n"
        summary += "\n"

        summary += "Diagnoses History (Unique):\n"
        unique_diagnoses = sorted(list(set(record["diagnosis"] for record in citizen.medical_records)))
        if unique_diagnoses:
            for diag in unique_diagnoses:
                summary += f"  - {diag}\n"
        else:
            summary += "  No diagnoses recorded.\n"
        summary += "\n"
        
        summary += "Active Prescriptions:\n"
        active_prescriptions = self.get_active_prescriptions(umn)
        if active_prescriptions:
            for p in active_prescriptions:
                summary += f"  - {p['medicine_name']} ({p['dosage']}, {p['frequency_per_day']}x/day, until {p['end_date']})\n"
        else:
            summary += "  No active prescriptions.\n"
        
        summary += "\n--- End of Summary ---\n"
        return summary

# --- Main Application Logic (Example Usage) ---
if __name__ == "__main__":
    system = WioAICareSystem()

    print("--- Wio AI Care System Demo ---")

    # 1. Register Citizens
    umn_alice = system.register_citizen("Alice Wonderland", "1990-05-15", "alice@example.com")
    umn_bob = system.register_citizen("Bob The Builder", "1985-11-20", "bob@example.com")
    umn_charlie = system.register_citizen("Charlie Brown", "2005-02-10", "charlie@example.com")
    system.register_citizen("Invalid DOB User", "2000/01/01") # Test invalid DOB

    if not umn_alice or not umn_bob or not umn_charlie:
        print("Failed to register one or more citizens. Exiting demo.")
        exit()
        
    print("\n--- Adding Medical Data ---")
    # 2. Add Medical Records
    system.add_medical_record(umn_alice, "2023-01-10", "Dr. Smith", "Common Cold", ["cough", "fever", "sore throat"], "Rest and fluids")
    system.add_medical_record(umn_alice, "2023-06-20", "Dr. Jones", "Allergic Rhinitis", "sneezing", "Antihistamines prescribed")
    system.add_medical_record(umn_alice, "2023-08-15", "Dr. Jones", "Follow-up Allergic Rhinitis", "sneezing improved", "Continue medication as needed")
    
    system.add_medical_record(umn_bob, "2023-03-01", "Dr. Health", "Hypertension Stage 1", "headache", "Lifestyle changes, monitor BP")
    system.add_medical_record(umn_bob, "2023-09-05", "Dr. Health", "Hypertension Follow-up", "BP slightly high", "Start Lisinopril 5mg")
    system.add_medical_record(umn_bob, "2023-09-15", "Dr. Cardio", "Minor Chest Pain - Ruled out MI", "chest pain", "Stress test scheduled")

    # 3. Add Prescriptions
    print("\n--- Adding Prescriptions ---")
    system.add_prescription(umn_alice, "Claritin", "10mg tablet", 1, "2023-06-20", 30) # Past
    system.add_prescription(umn_alice, "Amoxicillin", "250mg capsule", 3, datetime.date.today().strftime("%Y-%m-%d"), 7) # Current

    system.add_prescription(umn_bob, "Lisinopril", "5mg tablet", 1, "2023-09-05", 90) # Current
    system.add_prescription(umn_bob, "Aspirin", "81mg tablet", 1, "2023-09-15", 365) # Current, long term

    # 4. Get Summaries
    print("\n--- Medical Summaries ---")
    print(system.summarize_citizen_medical_data(umn_alice))
    print(system.summarize_citizen_medical_data(umn_bob))
    print(system.summarize_citizen_medical_data(umn_charlie)) # Charlie has no records

    # 5. AI Analysis (Simulated)
    print("\n--- AI Analysis ---")
    print(f"AI Insights for Alice (UMN: {umn_alice}):")
    for insight in system.ai_analyze_medical_data(umn_alice):
        print(f"  - {insight}")
    
    print(f"\nAI Insights for Bob (UMN: {umn_bob}):")
    for insight in system.ai_analyze_medical_data(umn_bob):
        print(f"  - {insight}")

    # 6. Medicine Reminders
    print("\n--- Medicine Reminders (Next 24 hours) ---")
    
    print(f"\nUpcoming reminders for Alice (UMN: {umn_alice}):")
    reminders_alice = system.get_upcoming_reminders(umn_alice, lookahead_hours=24)
    if reminders_alice:
        for r in reminders_alice:
            print(f"  - Take {r['medicine_name']} ({r['dosage']}) at {r['reminder_time'].strftime('%Y-%m-%d %H:%M:%S')}")
        # Simulate acknowledging the first reminder
        first_reminder_time_str = reminders_alice[0]['reminder_time'].strftime('%Y-%m-%d %H:%M:%S')
        system.acknowledge_reminder(umn_alice, first_reminder_time_str)
        print("  (First reminder acknowledged)")

        print(f"\nUpcoming reminders for Alice (UMN: {umn_alice}) after acknowledgment:")
        reminders_alice_after = system.get_upcoming_reminders(umn_alice, lookahead_hours=24)
        for r in reminders_alice_after:
             print(f"  - Take {r['medicine_name']} ({r['dosage']}) at {r['reminder_time'].strftime('%Y-%m-%d %H:%M:%S')}")

    else:
        print("  No upcoming reminders for Alice.")

    print(f"\nUpcoming reminders for Bob (UMN: {umn_bob}):")
    reminders_bob = system.get_upcoming_reminders(umn_bob, lookahead_hours=48) # Look further for Bob
    if reminders_bob:
        for r in reminders_bob:
            print(f"  - Take {r['medicine_name']} ({r['dosage']}) at {r['reminder_time'].strftime('%Y-%m-%d %H:%M:%S')}")
    else:
        print("  No upcoming reminders for Bob.")
    
    print("\n--- Demo End ---")
