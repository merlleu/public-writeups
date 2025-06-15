# Data Science
## 0. Introduction

```
Une entreprise parisienne a été victime d'une fuite de données sur un serveur accessible physiquement uniquement. 3 intrus ont mené une opération minutieuse tout au long de l'année. Pour ce faire, ils se sont chacun créé une fausse identité d'employé. Voici le log d'accès physique au bâtiment de l'entreprise, retrouvez les 3 identifiants suspects. PS : Il paraît que les identifiants des 3 intrus combinés forment une phrase cohérente...
````

We have a huge (408M !) csv called 'calendar.csv' that contains employee clock-in and clock-out data. The file has the following columns: `employee_id`, `type`, and `datetime`. The `type` column can either be 'in' or 'out', indicating whether the employee is clocking in or out.


```csv
employee_id,type,datetime
laurel,in,2024-01-01 08:45:01
yellowed,in,2024-01-01 08:45:01
audio,in,2024-01-01 08:45:05
landholdings,in,2024-01-01 08:45:06
```

At a quick glance, we see that the employees clock in between 8:45:00 and 9:15:59, then clock out between 16:45:00 and 17:05:59.

## 1. Employee 1:

#### Indice: La montre de cet employé doit être particulièrement rare...
```py
import pandas as pd

df = pd.read_csv("calendar.csv")
df['datetime'] = pd.to_datetime(df['datetime'])
df['date'] = df['datetime'].dt.date

daily_spans = (df.groupby(['employee_id', 'date'])['datetime']
               .agg(['min', 'max'])
               .reset_index())

daily_spans['duration'] = (daily_spans['max'] - daily_spans['min']).dt.total_seconds() / 3600

valid_days = daily_spans[(daily_spans['duration'] > 1) & (daily_spans['duration'] <= 16)]

stats = (valid_days.groupby('employee_id')['duration']
         .agg(['mean', 'std', 'count'])
         .query('count >= 5')
         .rename(columns={'mean': 'avg_duration', 'std': 'std_dev'})
         .sort_values('std_dev')
         .reset_index())

print(f"{'Employee ID':<15} {'Std Dev':<12} {'Avg Hours':<12} {'Count':<8}")
print("-" * 50)
for _, row in stats.head(10).iterrows():
    print(f"{row['employee_id']:<15} {row['std_dev']:<12.4f} {row['avg_duration']:<12.2f} {row['count']:<8.0f}")
```
The idea here is to calculate the average duration of presence for each employee, and then calculate the standard deviation of these durations.
Then sort the employees by their standard deviation, and print the top 10 most regular employees.

```bash
Employee ID     Std Dev      Avg Hours    Count   
--------------------------------------------------
finding         0.0341       8.00         217     
inevitability   0.1311       8.02         219     
lipoprotein     0.1331       8.00         224     
advances        0.1340       8.00         216     
exhaustion      0.1343       8.01         219     
xible           0.1344       8.00         223     
engineering     0.1345       8.00         219     
ileum           0.1349       7.98         223     
valorem         0.1352       7.99         214     
abode           0.1352       7.99         220 
```

Clearly, `finding` is the most regular employee by a large margin, with a standard deviation of 0.0341 hours (2 minutes) and an average of 8 hours worked per day over 217 days.
Flag: `AMSI{finding}`

## 2. Employee 2:

#### Indice: Cet employé aurait bien besoin de se reposer !


This one took me a while, but I observed that friday and saturday we have a lot less employees working.
So I decided to check if there were employees that worked every "high-attendance" day, which I defined as days with more than 5000 employees clocking in.

```py
import pandas as pd

df = pd.read_csv("calendar.csv")
df['datetime'] = pd.to_datetime(df['datetime'])
df['date'] = df['datetime'].dt.date

daily_counts = df.groupby('date')['employee_id'].nunique()
high_traffic_days = set(daily_counts[daily_counts > 5000].index)

if high_traffic_days:
    employee_days = df.groupby('employee_id')['date'].apply(set).reset_index()
    perfect_attendance = employee_days[employee_days['date'].apply(lambda x: high_traffic_days.issubset(x))]
    result = perfect_attendance['employee_id'].tolist()
else:
    result = []

print(result)
```
We check if someone worked every day where less than 5000 employees clocked in.

```bash
["hidden"]
```

Here we see that the employee `hidden` is the only one that worked every day with low attendance.
Flag: `AMSI{hidden}`

## 3. Employe 3 :

#### Mais comment cet employé fait-il pour être à deux endroits à la fois...
To be honest, I did get this one when trying to solve the first one.
It's quite easy we just have to find employees that do not clock in but only clock out.
```rust
use std::{collections::HashMap, error::Error};
use serde::Deserialize;

#[derive(Deserialize)]
struct Record {
    employee_id: String,
    event_type: String,
    datetime: String,
}

fn main() -> Result<(), Box<dyn Error>> {
    let mut employee_status: HashMap<String, bool> = HashMap::new();
    for result in csv::Reader::from_path("calendar.csv")?.records() {
        let record: Record = result?.deserialize(None)?;
        
        let is_in = record.event_type.as_str() == "in";
        if Some(&is_in) == employee_status.get(&record.employee_id) {
            println!("Inconsistent record for employee {} at {}", record.employee_id, record.datetime);
        }
        employee_status.insert(record.employee_id.clone(), is_in);
    }
    Ok(())
}
```

```bash
% cargo run -r
   Compiling employee_analyzer v0.1.0
    Finished `release` profile [optimized] target(s) in 0.30s
     Running `target/release/employee_analyzer`
Inconsistent record for employee anomalies at 2024-03-18 09:02:31
Inconsistent record for employee anomalies at 2024-03-20 16:59:09
Inconsistent record for employee anomalies at 2024-03-25 09:05:03
Inconsistent record for employee anomalies at 2024-03-27 16:56:16
Inconsistent record for employee anomalies at 2024-03-29 08:56:30
Inconsistent record for employee anomalies at 2024-04-03 17:05:29
Inconsistent record for employee anomalies at 2024-04-05 09:12:06
```

We see that the employee `anomalies` has clocked out without clocking in on multiple occasions.
Flag: `AMSI{anomalies}`

## Conclusion
This was a fun challenge, I really enjoyed working with the data and finding patterns in it.
The flags combined give us a nice message: `finding_hidden_anomalies`.
I personally think we the last flag should've been the combined flag because it was the easiest to find.