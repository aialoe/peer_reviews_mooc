# SQL Queries

There are 221,043 rows in the peer_submission_part_urls table

~~~
SELECT COUNT (*)
FROM peer_submission_part_urls
~~~

First, we want to join these URLs with some more information about the assignments in the peer_submissions table.  
We select only those rows that have a valid-looking URL (starting with 'http') AND have a score for the submission.  
Creates table with 42,253 rows (38,981 unique users).
~~~
CREATE TABLE scored_submission_urls
AS SELECT *
	FROM peer_submissions
	INNER JOIN peer_submission_part_urls USING (peer_submission_id, peer_assignment_id)
	WHERE peer_submission_score IS NOT NULL
	AND peer_submission_part_url_url LIKE 'http%';
~~~

There are no URLs associated with multiple distinct users (good).
~~~
SELECT *
FROM scored_submission_urls WHERE peer_submission_part_url_url IN
  (SELECT peer_submission_part_url_url FROM scored_submission_urls GROUP BY peer_submission_part_url_url HAVING COUNT(DISTINCT uva_peer_assignments_user_id) > 1)
  ORDER BY peer_submission_part_url_url DESC;
~~~

1,568 users have multiple, graded submissions with different URLs.
~~~
WITH X AS (SELECT * FROM scored_submission_urls WHERE uva_peer_assignments_user_id IN
  (SELECT uva_peer_assignments_user_id FROM scored_submission_urls GROUP BY uva_peer_assignments_user_id HAVING COUNT(DISTINCT peer_submission_part_url_url) > 1)
	ORDER BY uva_peer_assignments_user_id DESC)
SELECT COUNT (DISTINCT uva_peer_assignments_user_id)
FROM X
~~~

For now, let's just look at users with exactly one graded submission. 36,623 rows.

~~~
CREATE TABLE cleared_submissions
AS SELECT * 
FROM scored_submission_urls WHERE uva_peer_assignments_user_id IN
  (SELECT uva_peer_assignments_user_id
   FROM scored_submission_urls
   GROUP BY uva_peer_assignments_user_id
   HAVING COUNT(DISTINCT peer_submission_id) = 1)
~~~

---

# Additional cleaning steps

First the files must be downloaded

Before Downloading:
1. Only urls ending in ".pdf"
Students were explicitly required to submit pdfs.
37,413 --> 32,525 

At Download Time:
1. Only URLs that do not result in an HTTP or Connection Error
2. Filesize smaller than 10,000,000 bytes (roughly 10 MB)
Roughly 73 files excluded (I changed the logging procedure halfway through, not 100% sure)

At Parse Time:
1. Only PDF files that can be opened and parsed by the Fitz library
2. Only PDFs that are 4 pages long or fewer 
    Almost all submissions 5 pages or longer were either plagiarized or slidedecks that do not parse properly.
3. Only texts with at least 50 whitespace-delimited character sequences 
    An approximation of word count.
4. Only texts that are labelled as English by the CLD2 language detection library.
    CLD is developed by Google and used in the Chrome web browser. This removes non-English as well as mixed-language texts without penalizing spelling errors or character encoding errors.
Of the first 15,000, 1,915 files were excluded (almost 13%, mostly files that were too long).

The resulting batched_jsonls files contain 27,893  texts.

# Linking text to variables

Submissions are scored out of 18 and must be scored by at least 3 peers as per peer_assignments table

scored_submission_urls contains the following assignment versions:

| Assignment Version          | Count |
|-----------------------------|-------|
| "TBJA8aIfEeehoRK8dEj7bA@1"  | 102   |
| "3UBWMXdTEeWaSA7BQ_jXrQ@23" | 7434  |
| "3UBWMXdTEeWaSA7BQ_jXrQ@24" | 16732 |
| "3UBWMXdTEeWaSA7BQ_jXrQ@25" | 17413 |
| "3UBWMXdTEeWaSA7BQ_jXrQ@22" | 572   |

As per peer_assignment_review_schema_parts table, all versions are scored using the same six cateogires, 3 points each (18 points total):  
1. Challenge
2. Selection
3. Application
4. Insight
5. Approach
6. Organization

peer_reviews lists the submissions (identified by peer_submission_id) with all associated reviews (peer_review_id) and the user who made the review (uva_peer_assignments_user_id). peer_submission_part_scores lists the score assigned to each of the six categories (peer_submission_part_score, peer_assignment_review_schema_part_id) and can be tied back to a user via the peer_review_id. Text responses are also available under peer_review_part_free_responses. 

