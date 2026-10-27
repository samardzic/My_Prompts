## Read closet dimensions from the image:

- closet is comprised from 3 vertical segments 86cm wide each and 250 cm in height
- depth of the closet is 60cm 
- important: keep the drawing proportionate to size detected
- important: do calculations while reading the image, dimensions should correspond to actual height and width etc
- raise alert if the offset is more than 5cm



7 cm — bottom/plinth allowance


For the spacing im referring to the space in between the two surfaces to avoid friction (lets call it the "gap" which is 0.2cm by default).
Its a clearance between adjacent surfaces, not structural material thickness.

Vertical:
```text
5 + gap + 14 + gap + 14 + gap + 18 + gap + 18 + gap + 5
```

Horizontal:
```text
5 + gap + 72.5 + gap + 5 + gap + 72.5 + gap + 5
```


## Second prompt

1. Object type: Is this mainly for furniture/interior objects (chairs, cabinets, tables, appliances), or arbitrary objects?
2. Images: Will the user take the photos themselves, or will the tool accept any existing images?
3. Number of images: Do you expect, for example, 4–8 photos around the object, or an arbitrary number?
4. Camera angles: Should the tool require specific angles (front/back/left/right/top), or automatically determine the geometry?
5. Dimensions: Do you want the resulting OBJ to have real-world dimensions? If yes, how will the tool know the scale—e.g. user enters width = 160 cm?
6. Output: Only .obj, or also .mtl and texture images so the object can be imported into Sweet Home 3D with its appearance?
7. Accuracy: Is the goal a visually similar 3D model, or a dimensionally accurate model suitable for furniture/interior design?
8. Processing: Should this run locally on Windows, or can it use cloud/AI services?
9. Interface: Do you want a simple GUI, command-line tool, or both?
10. Sweet Home 3D: Is the ultimate goal specifically to create objects that can be imported directly into Sweet Home 3D's furniture library?


1. Its mainly for furniture/interior objects (chairs, cabinets, tables, appliances)
2. user will provide photos.
	- There would be project/sources folder
	- Within sources folder there would be ~3 images from different angles
	- app itself would guess dimensions based on proportions on the image. 
3. For source expect, 3–8 photos
4. tool needs to read angles from image name(front/back/left/right/top) or automatically determine the geometry.
	- Image name `front_blue_closet.jpg`
	- There could also be Image name `front_blue_closet_02.jpg` as a second source
	- There is `default_dimensions.json` file which will store some default values (fell free to suggest which)
5. Yes
6. .obj, and also .mtl and texture images so the object can be imported into Sweet Home 3D with its appearance
7. The the goal is a visually similar 3D model that can later be scaled as user see fit
8. Only locally. Keep in mind platform compatibility 
9. booth
10. The goal is to create objects that can be imported directly into Sweet Home 3D's furniture library, but if additional compatibility to some other tools is possible than do that also (eg blender ets)
