# EN3160 Assignment 02

**Fitting and Alignment**  
**Rahman MFA | 230507R**

Main notebook: `230507R_EN3160_A02.ipynb`

## Notebook sections
1. Sunflower blob detection using Laplacian of Gaussian (LoG) and scale-space extrema.
2. RANSAC line fitting followed by RANSAC circle fitting.
3. Homography-based flag projection onto planar surfaces.
4. PCB alignment using SIFT feature matching, RANSAC homography, perspective warping, and image differences.

## Folder setup
Place the provided Assignment 2 images in `Assignment_02/images/` using their original filenames:
- `the_berry_farms_sunflower_field.jpeg`
- `flag.png`
- `wadham.jpg`
- `AAAA1.jpg`
- `ADAD1.jpg`
- `AECC1.jpg`
- `AECC2.jpg`
- `AECC3.jpg`
- `AECC4.jpg`

The notebook uses paths relative to the Assignment 2 folder. The `outputs/` folder is created by the setup cell.

## Environment
Reuse the Python environment you used for Assignment 1, if available. Assignment 2 requires SciPy in addition to NumPy, Matplotlib and OpenCV:
```bash
pip install numpy matplotlib opencv-python scipy jupyter ipykernel
```

## Submission reminder
The report must be named `230507R_a02.pdf`, include the index number and name, and be **four pages or fewer**. Include key code, result images, comparisons, and a concise discussion. Commit and push work regularly.
