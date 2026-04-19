---
title: "What Does Validate Mean"
source: "https://dagster.io/glossary/data-validation"
author:
published:
created: 2026-04-20
description: "Learn what Validate means and how it fits into the world of data, analytics, or pipelines, all explained simply."
tags:
  - "clippings"
---
## Data validation definition:

Data validation is a critical step in the data engineering process to have trusted, complete data. Data validation can be performed at various stages, including during data ingestion, transformation, and loading.

The following are some best practices for data validation in the context of modern data pipelines:

- **Define clear validation rules:** It is essential to define clear validation rules to ensure that the data is validated correctly. These rules should be based on the data requirements, such as data type, format, and constraints.
- **Validate data as early as possible**: Data validation should be performed as early as possible in the data pipeline to detect errors as soon as possible. This approach can save time and resources by preventing the propagation of errors.
- **Use automated validation**: Automated validation can help ensure that the data is validated consistently and accurately. Automation can include using tools, scripts, or frameworks that check the data against predefined validation rules.
- **Monitor data quality:** Monitoring data quality is an ongoing process that involves tracking the data quality over time. This approach can help detect data quality issues early and take corrective actions.

## Data validation example using Python:

Please note that you need to [have the necessary Python libraries installed in your Python environment](https://dagster.io/glossary) to run this code.

Python provides several libraries and functions for data validation. Let's look at three of them:

**Pydantic**: Pydantic is a data validation library that provides runtime type checking and validation of data structures. Dagster uses Pydantic internally to validate data for configuration or resources, for example. Here’s an example of Pydantic in action:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str
    age: int

user_data = {"id": 1, "name": "John Doe", "email": "johndoe@example.com", "age": 30}
user = User(**user_data)
print(user)
```

This will output:

```
id=1 name='John Doe' email='johndoe@example.com' age=30
```

**Cerberus**: Cerberus is a lightweight and extensible data validation library that can validate complex data structures. Let’s look at an example of Cerberus in action:

```python
from cerberus import Validator

schema = {"name": {"type": "string"}, "age": {"type": "integer", "min": 18}}

data = {"name": "John Doe", "age": 25}

v = Validator(schema)
if v.validate(data):
    print("Data is valid")
else:
    print("Data is invalid")
    print(v.errors)
```

Output yielded by `v.errors`:

```javascript
Data is valid
```

**Great Expectations**: Great Expectations is a data validation library that can validate data quality and test data pipelines. It provides a range of features, including automated data profiling, data documentation, and testing data pipelines.

Here’s an example:

We can train Great Expecatations on a small data sample ( `small_dataset.csv` ), then apply the learnt rules to our large dataset ( `large_dataset.csv` ). The Small and Large datasets can be anything youlike, but should have a list of unique ids inthe `id` column:

You can use a sample as follows for the large dataset, and use just the first 10 rows in your small dataset.

```
id,Col A,Col B,Col C,Col D,Col E,Col F,Col G,Col H,Col I,Col J,Col K,Col L,Col M
1,8,5,8,10,7,0,0,5,2,8,9,4,9
2,5,8,4,3,10,6,8,6,2,2,2,4,7
3,9,8,5,10,6,3,1,2,5,5,7,3,5
[...]
996,8,8,5,4,3,8,5,9,9,7,7,5,0
997,8,6,6,7,6,5,5,4,7,7,0,5,8
998,1,9,8,1,4,7,10,3,6,5,4,1,4
999,1,5,1,1,4,0,1,5,5,5,1,2,1
1000,8,8,1,7,7,10,9,10,5,7,5,2,4
```

And apply this code to it:

```python
import great_expectations as ge

# Build up expectations on a representative sample of data and save them to disk
train = ge.read_csv("small_dataset.csv")
train.expect_column_values_to_not_be_null("id")
train.save_expectation_suite("my_expectations.json")

# Load in a new batch of data and test it against our expectations
test = ge.read_csv("large_dataset.csv")
validation_results = test.validate(expectation_suite="my_expectations.json")

# Take action based on the results
if validation_results["success"]:
    print ("The validation passed")
else:
    raise Exception("The validation failed")
```

This will output:

```javascript
The validation passed
```

So what is going on here?

1. We import the Great Expectations library
2. We use the \`ge.read\_csv()\`\` method to read in a CSV file containing a representative sample of data. In this case, the CSV file is named large\_dataset.csv.
3. We then use the \`expect\_column\_values\_to\_not\_be\_null()\`\` method to build an expectation that the "id" column in the data should not contain null values.
4. After building up our expectations, we use the `save_expectation_suite()` method to save them to disk in a JSON format. The resulting expectation suite is saved to a file named `my_expectations.json`.
5. Load in a new batch of data using the ge.read\_csv() method, again reading from the large\_dataset.csv file.
6. Use the `validate()` method to test the new data against our expectations. The `expectation_suite` parameter specifies the name of the JSON file containing our expectations.
7. The `validate()` method returns a dictionary of validation results. We check the "success" key of this dictionary to see if the validation passed or failed. If it passed, we print a message indicating that the validation was successful. If it failed, we raise an exception with a message indicating that the validation failed:
```python
Traceback (most recent call last):
  File "/Users/myname/validation-test.py", line 16, in <module>
    raise Exception("The validation failed")
Exception: The validation failed
```

Thanks to Ian Whitestone for [this example](https://ianwhitestone.work/hello-great-expectations/)