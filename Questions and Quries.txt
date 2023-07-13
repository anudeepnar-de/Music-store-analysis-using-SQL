--Q1: Who is the senior most employee based on job title?
SELECT * FROM employee
order by levels desc
limit 1;

--Q2: Which countries have the most Invoices?
SELECT * FROM invoice;

SELECT billing_country, COUNT(*) as total_invoices  FROM invoice
GROUP BY billing_country
ORDER BY total_invoices desc
LIMIT 1;
--- OR ---
SELECT billing_country, SUM(total) AS GT FROM invoice
group by billing_country
ORDER BY GT desc
LIMIT 3;

--Q3: What are top 3 values of total invoice?
SELECT * FROM invoice
ORDER BY total desc
LIMIT 3;

--Q4: Which country has the best customers? We would like to throw a promotional music festival in 
---the city we made the most money. Write a query that returns one city that has the higest sum of invoice 
---totals. Return both the city name & sum of all invoice total 
SELECT * FROM invoice;

SELECT billing_city, SUM(total) AS GT FROM invoice
group by billing_city
ORDER BY GT desc
LIMIT 1;

--Q5: Who is the best customer? The customer who has spent the most money will be declared the best customer.
---writr a query that returns the person who spent the most money.

SELECT customer.customer_id, customer.first_name, customer.last_name, SUM(invoice.total) as gt FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
GROUP BY customer.customer_id
ORDER BY gt DESC
LIMIT 1;

--Q6: Write query to return the email, first_name, last_name & Genre of all Rock Music listeners.
---Return your list ordered alphabetically by email starting with A

SELECT * FROM customer;
SELECT * FROM invoice;
SELECT * FROM invoice_line;
SELECT * FROM genre;

SELECT DISTINCT customer.email, customer.first_name, customer.last_name, (genre.name) FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
JOIN invoice_line ON invoice.invoice_id = invoice_line.invoice_id
JOIN track ON invoice_line.track_id = track.track_id
JOIN genre ON track.genre_id = genre.genre_id
where genre.name = 'Rock'
ORDER BY email;

------or-------

SELECT DISTINCT email, first_name, last_name FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
JOIN invoice_line ON invoice.invoice_id = invoice_line.invoice_id
WHERE track_id IN(
	SELECT track_id FROM track
	JOIN genre ON track.genre_id = genre.genre_id
	WHERE genre.name LIKE 'Rock'
)
ORDER BY email;

--Q7: Let's invite the artists who have written the most rock music in our dataset. Write a query
---that returns the Artist name and total track count of the top 10 rock bands. 

SELECT * FROM artist;
SELECT * FROM album;
SELECT * FROM track;
SELECT * FROM genre;


SELECT artist.artist_id, artist.name, COUNT(artist.artist_id) AS number_of_songs FROM track
JOIN album ON album.album_id = track.album_id
JOIN artist ON artist.artist_id = album.artist_id
JOIN genre ON genre.genre_id = track.genre_id
WHERE genre.name LIKE 'Rock'
GROUP BY artist.artist_id
ORDER BY number_of_songs DESC
LIMIT 10;

--Q8 Return all the track names that have a song length longer than the average song length. Return the 
-- Name and Milliseconds for each track. Order by the song length with the longest songs listed first.

SELECT name, milliseconds FROM track
WHERE milliseconds >(SELECT AVG(milliseconds) AS avg_track_length FROM track)
ORDER BY milliseconds DESC;

--Q9 Find how much amount spent by each customer on artists? write a query to return customer name, artist
--name and total spent

SELECT * FROM customer;
SELECT * FROM invoice;
SELECT * FROM artist;
SELECT * FROM invoice_line;

WITH best_selling_artist AS (
	SELECT artist.artist_id, artist.name AS artist_name, SUM(invoice_line.unit_price*invoice_line.quantity)
	FROM invoice_line
	JOIN track ON invoice_line.track_id = track.track_id
	JOIN album ON track.album_id = album.album_id
	JOIN artist ON album.artist_id = artist.artist_id
	GROUP BY artist.artist_id
	ORDER BY 3 DESC
	LIMIT 1
)
SELECT customer.customer_id, customer.first_name, customer.last_name, best_selling_artist.artist_name,
SUM(invoice_line.unit_price*invoice_line.quantity) AS amount_spent FROM invoice
JOIN customer ON customer.customer_id = invoice.customer_id
JOIN invoice_line ON invoice.invoice_id = invoice_line.invoice_id
JOIN track ON invoice_line.track_id = track.track_id
JOIN album ON track.album_id = album.album_id
JOIN best_selling_artist ON album.artist_id = best_selling_artist.artist_id
GROUP BY 1,2,3,4
ORDER BY 5 DESC;


--Q10 We want to find out the most popular music genre for each country. We determine the most popular genre 
--as the genre with the highest amout of purchases. Write that return each country along with the 
--top genre. For country where the maximum number of purchases is shared return all genres.

SELECT * FROM genre;
SELECT * FROM CUSTOMER;
SELECT DISTINCT customer.country FROM customer;

-- output 1
WITH temp_table AS (
	SELECT invoice_line.invoice_id,genre.genre_id, genre.name FROM track
	JOIN invoice_line ON track.track_id = invoice_line.track_id
	JOIN genre ON track.genre_id = genre.genre_id
)

SELECT customer.country, temp_table.genre_id, temp_table.name AS genre_name, COUNT(temp_table.name) FROM invoice
JOIN customer ON invoice.customer_id = customer.customer_id
JOIN temp_table ON temp_table.invoice_id = invoice.invoice_id
GROUP BY 1,2,3
ORDER BY customer.country, 4 DESC;

-- output 2

WITH popular_genre AS (
	SELECT COUNT(invoice_line.quantity) AS purchases, customer.country, genre.name, genre.genre_id,
	ROW_NUMBER() OVER(PARTITION BY customer.country ORDER BY COUNT(invoice_line.quantity)DESC) AS RowNo 
	FROM invoice_line
	JOIN invoice ON invoice.invoice_id = invoice_line.invoice_id
	JOIN customer ON customer.customer_id = invoice.customer_id
	JOIN track ON track.track_id = invoice_line.track_id
	JOIN genre ON genre.genre_id = track.genre_id
	GROUP BY 2,3,4
	ORDER BY 2 ASC, 1 DESC
)
SELECT * FROM popular_genre WHERE RowNo <=1 ;

--Q11 Write a query that determines the customer that has spent the most on music for each country. Write a query 
--that return the country along with top customer and how much they spent. For countries where the top amount 
--spent is shared, provide all customers who spent this amount.

SELECT * FROM customer;
SELECT * FROM invoice;

-- Method 1: using CTE 

WITH Customter_with_country AS (
		SELECT customer.customer_id, customer.first_name, customer.last_name, invoice.billing_country,SUM(invoice.total) AS total_spending,
	    ROW_NUMBER() OVER(PARTITION BY billing_country ORDER BY SUM(total) DESC) AS RowNo 
		FROM invoice
		JOIN customer ON customer.customer_id = invoice.customer_id
		GROUP BY 1,2,3,4
		ORDER BY 4 ASC,5 DESC)
SELECT * FROM Customter_with_country WHERE RowNo <= 1;


-- Method 2: Using Recursive 

WITH RECURSIVE 
	customter_with_country AS (
		SELECT customer.customer_id,first_name,last_name,billing_country,SUM(total) AS total_spending
		FROM invoice
		JOIN customer ON customer.customer_id = invoice.customer_id
		GROUP BY 1,2,3,4
		ORDER BY 2,3 DESC),

	country_max_spending AS(
		SELECT billing_country,MAX(total_spending) AS max_spending FROM customter_with_country
		GROUP BY billing_country)

SELECT cc.billing_country, cc.total_spending, cc.first_name, cc.last_name, cc.customer_id FROM customter_with_country cc
JOIN country_max_spending ms ON cc.billing_country = ms.billing_country
WHERE cc.total_spending = ms.max_spending
ORDER BY 1;

