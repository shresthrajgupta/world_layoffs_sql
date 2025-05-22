# Data Analysis using World Layoffs SQL Project

## Project Overview
This project uses the Layoffs 2022 dataset from Kaggle to analyze global tech layoffs using MySQL. The goal is to clean, transform, and perform exploratory data analysis (EDA) to extract meaningful insights from layoff trends across different companies, industries, and geographies.

*Dataset*: https://www.kaggle.com/datasets/swaptr/layoffs-2022

### Database Setup
```sql
CREATE DATABASE world_layoffs;
USE world_layoffs;

-- Import the dataset into a table named 'layoffs'

SELECT * FROM layoffs;
```

