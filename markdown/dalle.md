## Import Necessary Libaries and Define a Few Helper Functions


```python
import os
import openai
import itertools
import random
from PIL import Image
import requests
import uuid
import os
from PIL import Image
import matplotlib.pyplot as plt
import numpy as np
openai.api_key =  os.environ["OPENAI_API_KEY"]
openai.Model.list()
# Functions
def ask_gpt(prompt):
    response = openai.Completion.create(
        engine="text-davinci-003",  # use the latest model available to you
        prompt=prompt,
        max_tokens=200, 
    )
    output_text_parsed = response.choices[0].text.strip()
    print(output_text_parsed)
    return response
# make a uniqueish string to label images as they are generated
def generate_uniqueish_string():
    return str(uuid.uuid4())[:8]

def process_dalle_images(response, filename, image_dir, i, hash = True):
    # save the images
    uid = generate_uniqueish_string()
    urls = [datum["url"] for datum in response["data"]]  # extract URLs
    images = [requests.get(url).content for url in urls]  # download images
    image_names = [f"{filename}_{i + 1}_{uid}.png" for j in range(len(images))]  # create names
    filepaths = [os.path.join(image_dir, name) for name in image_names]  # create filepaths
    for image, filepath in zip(images, filepaths):  # loop through the variations
        with open(filepath, "wb") as image_file:  # open the file
            image_file.write(image)  # write the image to the file

    return filepaths
# set a directory to save DALL·E images to
image_dir_name = "images"
image_dir = os.path.join(os.curdir, image_dir_name)

# create the directory if it doesn't yet exist
if not os.path.isdir(image_dir):
    os.mkdir(image_dir)

# print the directory to save to
print(f"{image_dir=}")
def top_half_mask(width, height, mask_dir, mask_name):
    mask = Image.new("RGBA", (width, height), (0, 0, 0, 1))
    for x in range(width):
        for y in range(height // 2):
            mask.putpixel((x, y), (0, 0, 0, 0))
    mask_filepath = os.path.join(mask_dir, mask_name)
    mask.save(mask_filepath)

def bottom_half_mask(width, height, mask_dir, mask_name):
    mask = Image.new("RGBA", (width, height), (0, 0, 0, 1))
    for x in range(width):
        for y in range(height // 2, height):
            mask.putpixel((x, y), (0, 0, 0, 0))
    mask_filepath = os.path.join(mask_dir, mask_name)
    mask.save(mask_filepath)

def left_half_mask(width, height, mask_dir, mask_name):
    mask = Image.new("RGBA", (width, height), (0, 0, 0, 1))
    for x in range(width // 2):
        for y in range(height):
            mask.putpixel((x, y), (0, 0, 0, 0))
    mask_filepath = os.path.join(mask_dir, mask_name)
    mask.save(mask_filepath)

def right_half_mask(width, height, mask_dir, mask_name):
    mask = Image.new("RGBA", (width, height), (0, 0, 0, 1))
    for x in range(width // 2, width):
        for y in range(height):
            mask.putpixel((x, y), (0, 0, 0, 0))
    mask_filepath = os.path.join(mask_dir, mask_name)
    mask.save(mask_filepath)

```

    image_dir='./images'


## Show Image we are working with


```python
american_gothic = "images/American_Gothic_Square.png"
im = Image.open(american_gothic)
display(im)
```


    
![png](dalle_files/dalle_3_0.png)
    


## Use GPT-3 to generate a list of art styles to remix American Gothic


```python
# The question you want to ask GPT-3
question = "provide a python list of 15 distinct art styles (i.e. impressionist, cubist, pointlist, photorealistic, japanese wood block print)"
# Join the description and question into a single string
prompt = f"{question}"
gpt_output = ask_gpt(prompt)
# Parse the output
art_styles_string = gpt_output.choices[0]['text']
art_styles = [line.split('. ')[1] for line in art_styles_string.split('\n') if line]
# choose the length of combinations you want, for example 2
length_of_combinations = 2
style_combinations = list(itertools.combinations(art_styles, length_of_combinations))
```

    1. Impressionism 
    2. Cubism 
    3. Pointillism 
    4. Photorealism 
    5. Japanese Wood Block Print 
    6. Expressionism 
    7. Constructivism 
    8. Abstract Expressionism 
    9. Surrealism 
    10. Baroque 
    11. Realism 
    12. Neo-Impressionism 
    13. Art Deco 
    14. Cubo-Futurism 
    15. Op Art



```python
# Format the GPT output for Dalle prompt
formatted_strings = []

for combination in style_combinations:
    style_1, style_2 = combination
    formatted_string = f"in the style of {style_1} and {style_2}"
    formatted_strings.append(formatted_string)

for string in formatted_strings[:10]:
    print(string)

```

    in the style of Impressionism  and Cubism .
    in the style of Impressionism  and Pointillism .
    in the style of Impressionism  and Photorealism .
    in the style of Impressionism  and Japanese Wood Block Print .
    in the style of Impressionism  and Expressionism .
    in the style of Impressionism  and Constructivism .
    in the style of Impressionism  and Abstract Expressionism .
    in the style of Impressionism  and Surrealism .
    in the style of Impressionism  and Baroque .
    in the style of Impressionism  and Realism .


## Create Masks



```python
mask_dir = "images/masks"
# create the directory if it doesn't yet exist
if not os.path.isdir(mask_dir):
    os.mkdir(mask_dir)
# print the directory to save to
print(f"{mask_dir=}")

# TODO ask what are the actual image sizes 
width = 574
height = 574

mask_dir = "./masks"
os.makedirs(mask_dir, exist_ok=True)  # ensure the directory exists
top_half_mask(width, height, mask_dir, "top_half_mask.png")
bottom_half_mask(width, height, mask_dir, "bottom_half_mask.png")
left_half_mask(width, height, mask_dir, "left_half_mask.png")
right_half_mask(width, height, mask_dir, "right_half_mask.png")

# specify edit images dir
edit_image_dir = os.path.join("images", "edits")
```

    mask_dir='images/masks'



```python
os.makedirs(f"{edit_image_dir}", exist_ok=True)
```

## Call DALE API using a random mask of the 4 halves of the image, and a random combination of styles


```python
# TODO: find best way to add style tags to generated images in their metadata or just their filename

# Specify the directory
mask_dir = "./masks/"
num_iterations = 4  # specify the number of iterations

# Get the list of all masks in the directory
masks = [f for f in os.listdir(mask_dir) if f.endswith('.png')]

# Iterate for the number of specified iterations
for i in range(num_iterations):
    # Select a random mask
    selected_mask = random.choice(masks)
    # Get the full file path of the selected mask
    mask_filepath = os.path.join(mask_dir, selected_mask)

    # Select a random style combination
    selected_prompt = random.choice(formatted_strings)

    edit_response = openai.Image.create_edit(
        image=open(american_gothic, "rb"),  # local american gothic image
        mask=open(mask_filepath, "rb"),  # from right above
        prompt=selected_prompt,  # from the generation section
        n=1,
        size="512x512",
        response_format="url",
    )
    
    # print response for prototype / debug
    # print(edit_response)
    edit_filepaths = process_dalle_images(edit_response, "edits", edit_image_dir, i, hash = True)

```

## Plot the generated images in a collage


```python
# Specify the directory
image_directory = "images/edits/"
image_files = [f for f in os.listdir(image_directory) if f.endswith('.png')]
# shuffle image file order to get different images in the plot
image_files = np.random.permutation(image_files)
# Load all the images
images = [Image.open(image_directory + f) for f in image_files]
# If there are less than 100 images, repeat the list
while len(images) < 100:
    images *= 2

# Only take the first 100 images
images = images[:100]

# Create a 10x10 plot
fig, axes = plt.subplots(10, 10, figsize=(18, 18))

for i in range(10):
    for j in range(10):
        # Get the image
        img = images[i * 10 + j]

        # Remove the axes for each subplot
        axes[i, j].axis('off')

        # Display the image on the subplot
        axes[i, j].imshow(np.array(img), aspect='auto')

# Adjust the space between the subplots
# Negative values for wspace and hspace will make the images overlap
plt.subplots_adjust(wspace=-0.05, hspace=-0.05)

# Show the plot
plt.show()
```


    
![png](dalle_files/dalle_13_0.png)
    

