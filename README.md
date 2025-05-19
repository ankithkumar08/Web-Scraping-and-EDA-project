# Web-Scraping-and-EDA-project
# 📘 Project: Hotel Booking Data Analysis Across Indian States

# 🧩 Problem Statement
**To analyze hotel booking data across Indian states to explore pricing, discounts, customer ratings, and regional trends,**

**with the aim of providing insights for better travel decisions and hotel pricing strategies.**


**Objective:** 
- **To explore pricing trends and discount patterns across hotels in different states of India.**
- **To analyze customer ratings and reviews to understand factors affecting hotel preferences.**
- **To compare regional variations in hotel availability and pricing for travel planning and business insights.**


# importing the all the required libraries

```python
import requests
from bs4 import BeautifulSoup
import numpy as np
import pandas as pd
import re
```

# scraping the data form the website using the Beautifulsoup and the requests and Regular Expressions

```python
# List of all 28 Indian states
states = [
    "Andhra Pradesh", "Arunachal Pradesh","Assam", "Bihar", "Chhattisgarh", "Goa", "Gujarat", "Haryana", "Himachal Pradesh",
    "Jharkhand", "Karnataka", "Kerala", "Madhya Pradesh", "Maharashtra", "Manipur", "Meghalaya", "Mizoram", "Nagaland",
    "Odisha", "Punjab", "Rajasthan", "Sikkim", "Tamil Nadu", "Telangana", "Tripura", "Uttar Pradesh", "Uttarakhand", "West Bengal"
]
# Base URL structure
base_url = "https://www.booking.com/searchresults.en-gb.html"
checkin = "2025-07-31"
checkout = "2025-08-01"
params = f"&checkin={checkin}&checkout={checkout}&group_adults=2&no_rooms=1&group_children=0"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.85 Safari/537.36"
}

hotel_data = []

# Loop through each state
for state in states:
    formatted_state = state.replace(" ", "+")  # URL encode spaces as "+"
    url = f"{base_url}?ss={formatted_state}{params}"
    print(f"Scraping data for {state}...")
    
    # Loop through pages
    for offset in range(0, 100, 25):  # offset = 0, 25, 50,75 ...
        response = requests.get(url + f'&offset={offset}', headers=headers)
        soup = BeautifulSoup(response.content, "html.parser")
        
        hotels = soup.find_all('div', attrs={'data-testid': 'property-card'})
        
        if not hotels:
            print(f"No hotels found at offset {offset} for {state}. Ending scraping.")
            break
        
        for hotel in hotels:
            name_tag = hotel.find('div', attrs={'data-testid': 'title'})
            if name_tag:
                full_name = name_tag.get_text(strip=True)
                match = re.match(r'^[^,()\-\[\]{}\d]+', full_name)
                name = match.group(0).strip() if match else full_name
            else:
                name = None

            
            location_tag = hotel.find('span', attrs={'data-testid': 'address'})
            location = location_tag.get_text(strip=True) if location_tag else None
            
            original_price_tag = hotel.find('span', class_='fff1944c52 d68334ea31 ab607752a2')
            current_price_tag = hotel.find('span', attrs={'data-testid': 'price-and-discounted-price'})
            
            current_price = current_price_tag.get_text(strip=True).replace('\xa0', ' ') if current_price_tag else None
            original_price = original_price_tag.get_text(strip=True).replace('\xa0', ' ') if original_price_tag else current_price
            
            taxes_tag = hotel.find('div', {'data-testid': 'taxes-and-charges'})
            taxes_value = taxes_tag.get_text(strip=True) if taxes_tag else None
            
            review_score_tag = hotel.find('div', {'data-testid': 'review-score'})
            if review_score_tag:
                review_score = review_score_tag.get_text(strip=True)
                
                score_match = re.search(r"Scored\s+(\d+\.\d+)", review_score)
                rating_score = score_match.group(1) if score_match else None
                
                desc_match = re.search(r"\d+\.\d+\d+\.\d+([A-Za-z\s]+)\d", review_score)
                rating_desc = desc_match.group(1).strip() if desc_match else None
                
                reviews_match = re.search(r"(\d[\d,]*)\s+reviews", review_score)
                reviews = reviews_match.group(1) if reviews_match else None
            else:
                rating_score = None
                rating_desc = None
                reviews = None

            hotel_data.append({
                'Hotel Name': name,
                'Location': location,
                'Rating_Description': rating_desc,
                'Rating_Score': rating_score,
                'Reviews': reviews,
                'Original Price': original_price,
                'Current Price': current_price,
                'Taxes and Charges': taxes_value,
                'State': state
            })
        
        print(f"Scraped page with offset {offset} for {state}")
        

# Convert to DataFrame
df = pd.DataFrame(hotel_data)

# Show message and optionally save
print("Scraping completed successfully.")
```

```python
# df.to_csv("website_data.csv",index=False)
```

# Description:
**saving the scraped data into csv file**

```python
df.info()
```

```python
df.head()
```

```python
df.isnull().sum()/len(df)*100
```

**Some columns in the dataset have missing values (nulls).**

```python
df.duplicated().sum()
```

```python
df = df.drop_duplicates()
```

```python
df.duplicated().sum()
```

**To deal with missing values in the dataset, we used imputation methods:**


**ffill (forward fill): This fills missing values by carrying forward the last known value.**

**bfill (backward fill): This fills missing values using the next available value from below.**

```python
df = df.copy()
```

```python
df['Rating_Description']=df['Rating_Description'].fillna(df['Rating_Description'].bfill())
```

```python
df['Rating_Score']=df['Rating_Score'].fillna(df['Rating_Score'].bfill())
```

```python
df['Reviews']=df['Reviews'].fillna(df['Reviews'].ffill())

```

```python
df['Original Price']=df['Original Price'].fillna(df['Original Price'].bfill())
```

```python
df['Current Price']=df['Current Price'].fillna(df['Current Price'].bfill())
```

```python
df['Taxes and Charges']=df['Taxes and Charges'].fillna(df['Taxes and Charges'].bfill())
```

```python
df.isnull().sum()
```

**changing the datatypes of the columns**

```python
df['Original Price'] = df['Original Price'].str.replace(r'[₹,]', '', regex=True).astype(int)
```

```python
df['Current Price'] = df['Current Price'].str.replace(r'[₹,]', '', regex=True).astype(int)
```

```python
df['Taxes and Charges'] = (df['Taxes and Charges'].str.extract(r'(\d[\d,]*)').replace({None: '0'}).replace('', '0').replace(',', '', regex=True).astype(float))

```

```python
df['Reviews']=df['Reviews'].str.replace(',','').astype(int)
```

**creating a new columns based using the existing columns in the dataset**

```python
df['You Save (₹)'] =df['Original Price'] - df['Current Price']
```

```python
df['Discount %'] = round(((df['Original Price'] - df['Current Price']) / df['Original Price']) * 100,2)
```

```python
df['Total price'] = df['Current Price']+df['Taxes and Charges']
```

```python

```

```python

```

**saving the cleaned data into the local storage**

```python
# df.to_csv("final_dataset.csv",index=False)
```

```python
df=pd.read_csv("final_dataset.csv")
```

```python
df.info()
```

```python
df.head()
```

```python
df.isnull().sum()
```

```python
import matplotlib.pyplot as plt
import seaborn as sns
```

```python

```

# Selecting top 5 hotels with the highest discount percentages

```python
rating_counts = df['Rating_Description'].value_counts()

plt.pie(rating_counts, labels=rating_counts.index, autopct='%1.1f%%', startangle=140, colors=plt.cm.Pastel1.colors)
plt.title('Distribution of Rating Descriptions')
plt.tight_layout()
plt.show()
```

# Description of the Pie Chart:
- **This pie chart shows the Distribution of Rating Descriptions for hotels.**
- **The chart is divided into six segments representing different rating description**

- **Most hotels are rated between "Very Good" and "Good," indicating a generally positive customer perception.**
- **Only a small percentage of hotels are rated as "Exceptional," suggesting that truly outstanding hotels are rare in this dataset.**

```python
hotel_count_state = df['State'].value_counts()

# Create the bar chart
hotel_count_state.plot(kind='bar', color='skyblue', edgecolor='black')

for i, value in enumerate(hotel_count_state):
    plt.text(i, value + 1, str(value))

plt.title('Number of Hotels per State')
plt.xlabel('State')
plt.ylabel('Number of Hotels')
plt.xticks(rotation=90)
plt.tight_layout()

```

# Description of the Bar Chart:

- **This bar chart displays the Number of Hotels per State across different Indian states.**
- **Each bar represents a state, with the height of the bar corresponding to the number of hotels listed for that state. The exact number is also labeled on top of each bar for clarity.**
# Key Observations:
- **Goa has the highest number of hotels, with 66, followed closely by Andhra Pradesh (65) and Telangana (60).**
- **Odisha has the fewest hotels listed, with only 9.**

```python
rating_scores = df['Rating_Score'].dropna().astype(float)
sns.histplot(rating_scores, bins=20, kde=True, color='skyblue')

# Titles and labels
plt.title('Distribution of Rating Scores')
plt.xlabel('Rating Score')
plt.ylabel('Number of Hotels')
plt.tight_layout()
plt.show()

```

# Description of the Plot:

- **This histogram with a density curve shows the Distribution of Rating Scores for hotels.**
- **The x-axis represents the Rating Score ranging roughly from 1 to 10.**
- **The y-axis represents the Number of Hotels for each rating score bin.**
- **Most hotels have ratings clustered between 7 and 9, indicating that the majority of hotels are rated fairly high.**
- **The highest number of hotels fall around a rating of 8 to 8.5.**
- **There are very few hotels with ratings below 4 or above 10.**

```python
plt.figure(figsize=(8, 5))
sns.kdeplot(df['Current Price'], shade=True, color='green')
plt.title('KDE Plot of Current Hotel Prices')
plt.xlabel('Current Price (₹)')
plt.ylabel('Density')
plt.tight_layout()
plt.show()
```

# KDE Plot: Current Hotel Prices

**Most hotel prices are concentrated below ₹5,000.**

**The curve shows a right-skewed distribution, indicating a few hotels are much more expensive.**


```python
top_discounts = df.sort_values(by='Discount %', ascending=False).head(5)

# Create the bar chart
plt.figure(figsize=(10, 6))
bars = plt.bar(top_discounts['Hotel Name'], top_discounts['Discount %'], color='skyblue')
plt.title('Top 5 Hotels with Highest Discount Percentage')
plt.xlabel('Hotel Name')
plt.ylabel('Discount Percentage')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
```

# Description of the Plot:
- **This bar chart shows the Top 5 hotels that offer the highest discount percentages**.

- **The x-axis represents the hotel names.**

- **The y-axis represents the discount percentages these hotels are offering.**

```python
#Correlation between Original Price and Current Price
correlation = df['Original Price'].corr(df['Current Price'])
print(f"Correlation between Original Price and Discount %: {correlation:.2f}")

plt.scatter(df['Original Price'], df['Current Price'], color='skyblue', alpha=0.7)
plt.title('Correlation between Original Price and Current Price')
plt.xlabel('Original Price (₹)')
plt.ylabel('Current Price')
plt.grid(True)
plt.tight_layout()
plt.show()
```

# Description of the Plot:
- **This scatter plot shows the relationship between the Original Price and the Current Price of hotels.**
- **Each point represents a hotel, where the x-axis is its original listed price and the y-axis is the price it is currently offered at.**

```python
# Calculate average discount percentage per state
avg_discount_state = df.groupby('State')['Discount %'].mean().sort_values(ascending=False)

avg_discount_state.plot(kind='bar', color='skyblue')
plt.title('Average Discount Percentage per State')
plt.xlabel('State')
plt.ylabel('Average Discount (%)')
plt.xticks(rotation=90, ha='right')
plt.tight_layout()
plt.show()
```

# Description of the Plot:

- **This bar chart represents the Average Discount Percentage per State for hotels.**
- **States like Haryana, Telangana, and Maharashtra have the highest average discounts, with values close to or above 30%.**
- **On the other hand, states like Mizoram, Odisha, and Nagaland have the lowest average discounts, close to 0-2%.**

```python
# Scatter plot to visualize the relationship

df.plot.scatter(x='Rating_Score', y='Reviews', alpha=0.5)
plt.title('Rating Score vs. Number of
Reviews')
plt.xlabel('Rating Score')
plt.ylabel('Number of Reviews')
plt.xticks(rotation=90, ha='right')
plt.show()
```

# Description of the Scatter Plot:

**This scatter plot visualizes the relationship between Rating Scores and the Number of Reviews for hotels.**
**The x-axis represents the Rating Score, ranging from 1 to 10.**
**The y-axis shows the Number of Reviews, with values going up to over 2000.***
# Key Observations:
**The majority of hotels have rating scores clustered between 7 and 9, indicating generally favorable reviews.**
**Hotels with higher ratings tend to receive more reviews, suggesting a positive correlation between a hotel's rating and its popularity.**

```python
# Sort and select top 10 hotels
top_tax_hotels = df[['Hotel Name', 'Taxes and Charges']].sort_values(by='Taxes and Charges', ascending=False).head(10)

# Plot
plt.figure(figsize=(12, 6))
plt.bar(top_tax_hotels['Hotel Name'], top_tax_hotels['Taxes and Charges'], color='pink',edgecolor='black')
plt.title('Top 10 Hotels charging high texes')
plt.xlabel('Hotel Name')
plt.ylabel('Tax Percentage (%)')
plt.xticks(rotation=90, ha='right')
plt.tight_layout()
plt.show()
```

# Description of the Bar Chart:

- **This bar chart displays the Top 10 Hotels Charging High Taxes, based on their tax percentage (%).**
- **Each bar represents a hotel, with the height corresponding to the total tax percentage charged. The hotels are arranged in descending order of tax percentage.**
# Key Observations:
- **Tech Park Residency charges the highest tax percentage, significantly higher than others, at over 8000%.**
-**Taj Lake Palace Udaipur and The Leela Palace Udaipur follow, charging approximately 6300% and 5600% respectively.**

```python
# Top 10 hotels with the most reviews
top_reviewed_hotels = df.sort_values(by='Reviews', ascending=False)[['Hotel Name', 'Reviews']].head(10)
print(top_reviewed_hotels)
```

```python
plt.figure(figsize=(12, 6))
sns.barplot(x='Reviews',y='Hotel Name',data=top_reviewed_hotels,palette='magma')
plt.title('Top 10 Hotels with the Most Reviews')
plt.xlabel('Number of Reviews')
plt.ylabel('Hotel Name')
plt.tight_layout()
plt.show()
```

# Description of the Bar Chart:
- **This chart shows the Top 10 Hotels with the Most Reviews. It visually ranks hotels by the number of reviews received, with the values increasing from left to right.**
- **Each bar represents a hotel, and the length of the bar corresponds to the number of customer reviews.**

# Key Observations:
- **Hyatt Regency Amritsar Hotel & Spa tops the list with the highest number of reviews, slightly exceeding 2200.**
- **The Red Martin Near Jallianwala Bagh and Zulu Land Cottages follow closely, both also nearing the 2200 mark.**
- **The Oberoi Bengaluru ranks fourth, with reviews slightly above 2100.**

```python
# Top 5 Expensive Hotels
```

```python
plt.figure(figsize=(10, 6))
sns.boxplot(df,x='State', y='Discount %')
plt.title('Discount % Distribution by State')
plt.xlabel('State')
plt.ylabel('Discount %')
plt.xticks(rotation=90)
plt.tight_layout()
plt.show()

```

# Box Plot: Discount % Distribution by State
- **Shows how hotel discounts vary across different states.**

- **Wide variation in discounts, with some states offering up to 70–80% off.**

- **States like Goa, Maharashtra, Telangana and West Bengal show higher median discounts.**

- **Several states have low or no discounts, indicating less aggressive pricing strategies.**


```python
top_5_price = df.sort_values(by='Total price', ascending=False).head(5)
print(top_5_price[['Hotel Name', 'Total price']])

# Visualization
plt.figure(figsize=(10, 6))
sns.barplot(x='Total price', y='Hotel Name', data=top_5_price)
plt.title('Top 5 Expensive Hotels')
plt.xlabel('Price')
plt.ylabel('Hotel Name')
plt.tight_layout()
plt.show()

```

```python
plt.figure(figsize=(12, 6))
sns.violinplot(x='Rating_Description', y='Current Price', data=df)
plt.title('Price Distribution by Rating Description')
plt.xlabel('Rating Description')
plt.ylabel('Current Price (₹)')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

# Violin Plot: Price Distribution by Rating
- **Visualizes how hotel prices (₹) vary across different rating categories using a violin plot.**

- **"Exceptional", "Superb", and "Review score" ratings show the widest price range, reaching up to ₹50,000.
"Good" and "Very good" hotels generally have lower and more compact price ranges (₹1,000–₹6,000).**
- **Most hotel prices across categories are concentrated between ₹2,000–₹5,000.**


```python
plt.figure(figsize=(12, 6))
avg_price = df.groupby('Hotel Name')['Current Price'].mean().sort_values()
avg_price.plot(kind='line', marker='*')
plt.title('Average Current Price per Hotel')
plt.xlabel('Hotel Name')
plt.ylabel('Average Current Price (₹)')
plt.xticks(rotation=90)
plt.grid(True)
plt.tight_layout()
plt.show()

```

# Description of the Chart:
# Title: Top 5 Expensive Hotels
- **This chart displays the top 5 most expensive hotels based on their pricing.**
- **Each bar represents a hotel, and the length of the bar corresponds to the price (presumably in Indian Rupees).**

# Key Observations:
- **Tech Park Residency is the most expensive hotel on the list, with a price exceeding ₹53,000, significantly higher than the others.**

```python
top_per_state = df.loc[df.groupby('State')['Total price'].idxmax()]
print(top_per_state[['State', 'Hotel Name', 'Total price']])

# Visualization
plt.figure(figsize=(12, 6))
sns.barplot(x='State', y='Total price', hue='Hotel Name', data=top_per_state)
plt.title('Top Hotel in Each State by Price')
plt.xticks(rotation=90)
plt.tight_layout()
plt.show()

```

# Description of the Bar Chart:
**This is a bar chart titled "Top Hotel in Each State by Price", which visualizes the most expensive hotel in each Indian state, based on their total price.**


# Key Observations:
- **Karnataka (Tech Park Residency) shows the highest hotel price, exceeding ₹53,000.**
- **Goa (JW Marriott Goa) also stands out with a price above ₹24,000.**
- **Gujarat, Haryana, and Assam follow closely with hotels in the ₹18,000–21,000 range.**
- **States like Bihar, Mizoram, Nagaland, and Tripura show the lowest top hotel prices, typically below ₹10,000.**

```python
top_rated = df.loc[df.groupby('State')['Rating_Score'].idxmax()]
#print(top_rated[['State', 'Hotel Name', 'Rating_Score']])

# Visualization
sns.barplot(x='State', y='Rating_Score',  hue='Hotel Name',data=top_rated)
plt.title('Top Rated Hotel in Each State')
plt.xticks(rotation=90)
plt.tight_layout()
plt.show()
```

```python
cheapest_5 = df.nsmallest(5, 'Total price')
cheapest_5[['Hotel Name', 'Total price', 'State']]
```

```python
plt.figure(figsize=(10, 6))
sns.barplot(cheapest_5, x='Hotel Name', y='Total price', hue='State')
plt.title('Top 5 Cheapest Hotels')
plt.xlabel('Hotel Name')
plt.ylabel('Total Price (₹)')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

# Description of the Chart:
# Title: Top 5 Cheapest Hotels
- **This chart displays the top 5 most Cheapest Hotels based on their pricing.**
- **Each bar represents a hotel, and the length of the bar corresponds to the price (presumably in Indian Rupees).**

# Key Observations:
- **Uma homestay is the Cheapest Hotels on the list, with a price 84.0	, significantly lower than the others.**

```python
hotel_counts = df.groupby(['State', 'Location']).size().reset_index(name='Count')
most_hotels_location = hotel_counts.loc[hotel_counts.groupby('State')['Count'].idxmax()]
print(most_hotels_location)
```

```python
plt.figure(figsize=(12, 6))
sns.barplot(x='State', y='Count', hue='Location', data=most_hotels_location)
plt.title('Most Frequent Hotel Locations per State')
plt.xticks(rotation=90)
plt.tight_layout()
plt.show()
```

# Description of the Chart:
- **This bar chart is titled "Most Frequent Hotel Locations per State", and it visualizes which cities or towns within each Indian state have the highest number of hotels listed.**

# Key Observations:
- **Karnataka (Bangalore) has the highest count at 44 hotels.**
- **Punjab (Amritsar) and Rajasthan (Udaipur) also show high frequencies, with 40 and 39 hotels respectively.**

```python
plt.figure(figsize=(10, 8))
numerical_cols = ['Rating_Score', 'Reviews', 'Original Price', 'Current Price', 'Taxes and Charges', 'You Save (₹)', 'Discount %', 'Total price']
correlation_matrix = df[numerical_cols].corr()
sns.heatmap(correlation_matrix, annot=True, cmap='coolwarm', linewidths=0.5)
plt.title('Correlation Heatmap of Numerical Features')
plt.tight_layout()
plt.show()
```

# 📊 Correlation Analysis (Heatmap)
- **Analyzed relationships between ratings, reviews, prices, discounts, and taxes using a heatmap.**
- **Found strong positive correlation among:**
Original Price, Current Price, Taxes, and Total Price
- **Rating Score and Review Count showed weak correlation with pricing-related features.**


## Connect with Me 🤝
Find out more about my journey and connect with me on [LinkedIn](https://www.linkedin.com/in/ankithkumar08-data-analyst).

---

## Thank you for checking out my repository! Let’s code together! 💻
