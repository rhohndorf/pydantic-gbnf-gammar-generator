# Pydantic GBNF Grammar Generator

Pydantic GBNF Grammar Generator facilitates the conversion of Pydantic data models into GBNF grammars.
This library was created to use it in combination with llama.cpp and is at this point in fact just a repackaged version of some Python scripts contained in the [llama.cpp repository](https://github.com/ggerganov/llama.cpp) to make integration into other projects easier.

## Installation
The easiest way to install is from PYPI
```shell
pip install pydantic-gbnf-grammar-generator
```

Alternatively, you can install from source
```shell
git clone https://github.com/rhohndorf/pydantic-gbnf-gammar-generator.git
cd pydantic-gbnf-gammar-generator
pip install -e .
```

## Usage
The following example demonstrates the technical usage of the library. All examples can be found in the examples folder.
To run the examples a llama.cpp server listing on port 8080 is required.

### Structured Output

```Python
from enum import Enum
import json
from typing import Optional, List

from pydantic import BaseModel, Field
import requests

from pydantic_gbnf_grammar_generator import generate_gbnf_grammar_and_documentation

# Function to get completion on the llama.cpp server with grammar. 
def create_completion(prompt, grammar):
    headers = {"Content-Type": "application/json"}
    data = {"prompt": prompt, "grammar": grammar, "stop": ["<|im_end|>"]}

    response = requests.post("http://127.0.0.1:8080/completion", headers=headers, json=data)
    data = response.json()

    print(data["content"])
    return data["content"]

# A example structured output based on pydantic models. The LLM will create an entry for a Book database out of an unstructured text.
class Category(Enum):
    """
    The category of the book.
    """
    Fiction = "Fiction"
    NonFiction = "Non-Fiction"


class Book(BaseModel):
    """
    Represents an entry about a book.
    """
    title: str = Field(..., description="Title of the book.")
    author: str = Field(..., description="Author of the book.")
    published_year: Optional[int] = Field(..., description="Publishing year of the book.")
    keywords: List[str] = Field(..., description="A list of keywords.")
    category: Category = Field(..., description="Category of the book.")
    summary: str = Field(..., description="Summary of the book.")


# We need no additional parameters other than our list of pydantic models.
gbnf_grammar, documentation = generate_gbnf_grammar_and_documentation([Book])
print(gbnf_grammar)

system_message = "You are an advanced AI, tasked to create a dataset entry in JSON for a Book. The following is the expected output model:\n\n" + documentation

text = """The Feynman Lectures on Physics is a physics textbook based on some lectures by Richard Feynman, a Nobel laureate who has sometimes been called "The Great Explainer". The lectures were presented before undergraduate students at the California Institute of Technology (Caltech), during 1961–1963. The book's co-authors are Feynman, Robert B. Leighton, and Matthew Sands."""
prompt = f"<|im_start|>system\n{system_message}<|im_end|>\n<|im_start|>user\n{text}<|im_end|>\n<|im_start|>assistant"

text = create_completion(prompt=prompt, grammar=gbnf_grammar)

json_data = json.loads(text)

print(Book(**json_data))
```

### Function Calling

```Python
from enum import Enum
import json
from typing import Union

from pydantic import BaseModel, Field
import requests

from pydantic_gbnf_grammar_generator import generate_gbnf_grammar_and_documentation


# Function to get completion on the llama.cpp server with grammar.
def create_completion(prompt, grammar):
    headers = {"Content-Type": "application/json"}
    data = {"prompt": prompt, "grammar": grammar, "stop": ["<|im_end|>"]}

    response = requests.post("http://127.0.0.1:8080/completion", headers=headers, json=data)
    data = response.json()

    print(data["content"])
    return data["content"]


# A function for the agent to send a message to the user.
class SendMessageToUser(BaseModel):
    """
    Send a message to the User.
    """

    chain_of_thought: str = Field(..., description="Your chain of thought while sending the message.")
    message: str = Field(..., description="Message you want to send to the user.")

    def run(self):
        print(self.message)


# Enum for the calculator tool.
class MathOperation(Enum):
    ADD = "add"
    SUBTRACT = "subtract"
    MULTIPLY = "multiply"
    DIVIDE = "divide"


# Simple pydantic calculator tool for the agent that can add, subtract, multiply, and divide. Docstring and description of fields will be used in system prompt.
class Calculator(BaseModel):
    """
    Perform a math operation on two numbers.
    """

    number_one: Union[int, float] = Field(..., description="First number.")
    operation: MathOperation = Field(..., description="Math operation to perform.")
    number_two: Union[int, float] = Field(..., description="Second number.")

    def run(self):
        if self.operation == MathOperation.ADD:
            return self.number_one + self.number_two
        elif self.operation == MathOperation.SUBTRACT:
            return self.number_one - self.number_two
        elif self.operation == MathOperation.MULTIPLY:
            return self.number_one * self.number_two
        elif self.operation == MathOperation.DIVIDE:
            return self.number_one / self.number_two
        else:
            raise ValueError("Unknown operation.")


# Here the grammar gets generated by passing the available function models to generate_gbnf_grammar_and_documentation function. This also generates a documentation usable by the LLM.
# pydantic_model_list is the list of pydanitc models
# outer_object_name is an optional name for an outer object around the actual model object. Like a "function" object with "function_parameters" which contains the actual model object. If None, no outer object will be generated
# outer_object_content is the name of outer object content.
# model_prefix is the optional prefix for models in the documentation. (Default="Output Model")
# fields_prefix is the prefix for the model fields in the documentation. (Default="Output Fields")
gbnf_grammar, documentation = generate_gbnf_grammar_and_documentation(
    pydantic_model_list=[SendMessageToUser, Calculator],
    outer_object_name="function",
    outer_object_content="function_parameters",
    model_prefix="Function",
    fields_prefix="Parameters",
)

print(gbnf_grammar)
print(documentation)

system_message = (
    "You are an advanced AI, tasked to assist the user by calling functions in JSON format. The following are the available functions and their parameters and types:\n\n"
    + documentation
)

user_message = "What is 42 * 42?"
prompt = (
    f"<|im_start|>system\n{system_message}<|im_end|>\n<|im_start|>user\n{user_message}<|im_end|>\n<|im_start|>assistant"
)

text = create_completion(prompt=prompt, grammar=gbnf_grammar)
function_dictionary = json.loads(text)
if function_dictionary["function"] == "Calculator":
    function_parameters = {**function_dictionary["function_parameters"]}

    print(Calculator(**function_parameters).run())
    # This should output: 1764

```