# TTB Public Comments via Regulations.gov

```js
const comments_file = FileAttachment("./data/public_comments.json").json();
```

```js
// Load the actual comments data
const comments = await comments_file;
```

```js
// Get file metadata for display with error handling
const commentsFile = FileAttachment("./data/public_comments.json");
const fileSize = commentsFile.size ? (commentsFile.size / 1024).toFixed(1) : "Unknown"; // Convert to KB
const lastModified = commentsFile.lastModified ? new Date(commentsFile.lastModified) : null;
const href = commentsFile.href;
const downloadName = "public_comments_" + lastModified.toISOString().replace(/[:.]/g, "-") + ".json";
```

<div style="margin-bottom: 1rem;">
  <small style="color: #666; font-size: 0.75em;">
    Data as of ${lastModified.toLocaleString()} | <a href="./data/public_comments.json" download="public_comments.json" style="color: #0066cc;">Download raw data</a>
  </small>
</div>

```js
// Search functionality
const searchInput = Inputs.text({
  placeholder: "Search across all tables (document titles, agencies, countries, etc.)",
  width: 600
});

const searchTerm = Generators.input(searchInput);
```

<div style="margin-bottom: 1rem;">
  ${searchInput}
</div>

```js
// Apply single filtering across all data based on search term
const filteredComments = searchTerm ? 
  comments.filter(d => 
    (d.document_title && d.document_title.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.document_type && d.document_type.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.docket_id && d.docket_id.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.gov_agency_type && d.gov_agency_type.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.gov_agency && d.gov_agency.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.country && d.country.toLowerCase().includes(searchTerm.toLowerCase())) ||
    (d.comment_text && d.comment_text.toLowerCase().includes(searchTerm.toLowerCase()))
  ) : comments;
```


```js
// Group comments by document and get all documents with comment counts
const documentSummary = Array.from(
  d3.rollup(
    filteredComments.filter(d => d.document_posted_date), // Only include documents with valid posted dates
    v => {
      // Calculate count of comments with any attachments for this document
      const commentsWithAttachments = v.filter(d => (d.attachment_count || 0) > 0).length;
      
      // Find the most recent comment date for this document
      const commentDates = v.map(d => d.comment_received_date || d.comment_posted_date).filter(d => d);
      const lastCommentDate = commentDates.length > 0 ? 
        new Date(Math.max(...commentDates.map(d => new Date(d)))) : null;
      
      return {
        comment_count: v.length,
        document_title: v[0].document_title || "Unknown Title",
        document_type: v[0].document_type || "Unknown",
        document_posted_date: v[0].document_posted_date,
        docket_id: v[0].docket_id || "Unknown",
        comments_with_attachments: commentsWithAttachments,
        last_comment_date: lastCommentDate
      };
    },
    d => d.document_id
  ),
  ([document_id, data]) => ({
    document_id,
    ...data
  })
)
.sort((a, b) => {
  // Sort by most recent last comment date first, with null dates at the end
  if (!a.last_comment_date && !b.last_comment_date) return 0;
  if (!a.last_comment_date) return 1;
  if (!b.last_comment_date) return -1;
  return new Date(b.last_comment_date) - new Date(a.last_comment_date);
}); // Sort by most recent last comment first

// Show detailed table of all documents
display(html`<h4>All Documents Detail ${searchTerm ? `(filtered: ${documentSummary.length} documents from ${filteredComments.length} comments)` : `(${documentSummary.length} total documents)`}</h4>`);
display(Inputs.table(documentSummary, {
  columns: [
    "document_title",
    "comment_count", 
    "comments_with_attachments",
    "document_posted_date",
    "last_comment_date",
    "document_type",
    "docket_id"
  ],
  header: {
    document_title: "Document Title",
    comment_count: "Comments",
    comments_with_attachments: "W/ Attachments",
    document_posted_date: "Posted Date",
    last_comment_date: "Last Comment",
    document_type: "Type",
    docket_id: "Docket ID"
  },
  format: {
    document_posted_date: d => new Date(d).toLocaleDateString(),
    last_comment_date: d => d ? new Date(d).toLocaleDateString() : "N/A",
    comment_count: d => d.toLocaleString(),
    comments_with_attachments: d => d.toLocaleString()
  },
  width: {
    document_title: 350,
    comment_count: 80,
    comments_with_attachments: 90,
    document_posted_date: 100,
    last_comment_date: 100,
    document_type: 120,
    docket_id: 120
  }
}));
```

```js
// Group comments by government agency type and agency combination
const agencySummary = Array.from(
  d3.rollup(
    filteredComments,
    v => {
      // Calculate unique documents for this agency combination
      const uniqueDocuments = new Set(v.map(d => d.document_id)).size;
      
      // Calculate comments with attachments
      const commentsWithAttachments = v.filter(d => (d.attachment_count || 0) > 0).length;
      
      // Calculate total attachments
      const totalAttachments = d3.sum(v, d => d.attachment_count || 0);
      
      // Get date range
      const commentDates = v.map(d => d.comment_received_date || d.comment_posted_date).filter(d => d);
      const earliestComment = commentDates.length > 0 ? 
        new Date(Math.min(...commentDates.map(d => new Date(d)))) : null;
      const latestComment = commentDates.length > 0 ? 
        new Date(Math.max(...commentDates.map(d => new Date(d)))) : null;
      
      return {
        comment_count: v.length,
        document_count: uniqueDocuments,
        comments_with_attachments: commentsWithAttachments,
        total_attachments: totalAttachments,
        attachment_rate: v.length > 0 ? (commentsWithAttachments / v.length * 100) : 0,
        earliest_comment: earliestComment,
        latest_comment: latestComment
      };
    },
    d => d.gov_agency_type || "Unknown Type",
    d => d.gov_agency || "Unknown Agency"
  ),
  ([gov_agency_type, agencyMap]) =>
    Array.from(agencyMap, ([gov_agency, stats]) => ({
      gov_agency_type,
      gov_agency,
      ...stats
    }))
).flat()
.sort((a, b) => b.comment_count - a.comment_count); // Sort by most comments first

// Show agency analysis table
display(html`<h4>Government Agency Analysis ${searchTerm ? `(${agencySummary.length} agency combinations from ${filteredComments.length} comments)` : `(${agencySummary.length} total combinations)`}</h4>`);
display(Inputs.table(agencySummary, {
  columns: [
    "gov_agency_type",
    "gov_agency",
    "comment_count",
    "document_count",
    "comments_with_attachments",
    "attachment_rate",
    "total_attachments",
    "earliest_comment",
    "latest_comment"
  ],
  header: {
    gov_agency_type: "Agency Type",
    gov_agency: "Agency",
    comment_count: "Comments",
    document_count: "Documents",
    comments_with_attachments: "W/ Attachments",
    attachment_rate: "Attach Rate %",
    total_attachments: "Total Attachments",
    earliest_comment: "First Comment",
    latest_comment: "Last Comment"
  },
  format: {
    comment_count: d => d.toLocaleString(),
    document_count: d => d.toLocaleString(),
    comments_with_attachments: d => d.toLocaleString(),
    attachment_rate: d => `${d.toFixed(1)}%`,
    total_attachments: d => d.toLocaleString(),
    earliest_comment: d => d ? new Date(d).toLocaleDateString() : "N/A",
    latest_comment: d => d ? new Date(d).toLocaleDateString() : "N/A"
  },
  width: {
    gov_agency_type: 120,
    gov_agency: 200,
    comment_count: 80,
    document_count: 80,
    comments_with_attachments: 90,
    attachment_rate: 80,
    total_attachments: 100,
    earliest_comment: 100,
    latest_comment: 100
  }
}));
```

```js
// Group comments by country
const countrySummary = Array.from(
  d3.rollup(
    filteredComments,
    v => {
      // Calculate unique documents for this country
      const uniqueDocuments = new Set(v.map(d => d.document_id)).size;
      
      // Calculate comments with attachments
      const commentsWithAttachments = v.filter(d => (d.attachment_count || 0) > 0).length;
      
      // Calculate total attachments
      const totalAttachments = d3.sum(v, d => d.attachment_count || 0);
      
      // Get date range
      const commentDates = v.map(d => d.comment_received_date || d.comment_posted_date).filter(d => d);
      const earliestComment = commentDates.length > 0 ? 
        new Date(Math.min(...commentDates.map(d => new Date(d)))) : null;
      const latestComment = commentDates.length > 0 ? 
        new Date(Math.max(...commentDates.map(d => new Date(d)))) : null;
      
      return {
        comment_count: v.length,
        document_count: uniqueDocuments,
        comments_with_attachments: commentsWithAttachments,
        total_attachments: totalAttachments,
        attachment_rate: v.length > 0 ? (commentsWithAttachments / v.length * 100) : 0,
        earliest_comment: earliestComment,
        latest_comment: latestComment
      };
    },
    d => d.country || "Unknown"
  ),
  ([country, stats]) => ({
    country,
    ...stats
  })
)
.sort((a, b) => b.comment_count - a.comment_count); // Sort by most comments first

// Show country analysis table
display(html`<h4>Country Analysis ${searchTerm ? `(${countrySummary.length} countries from ${filteredComments.length} comments)` : `(${countrySummary.length} total countries)`}</h4>`);
display(Inputs.table(countrySummary, {
  columns: [
    "country",
    "comment_count",
    "document_count",
    "comments_with_attachments",
    "attachment_rate",
    "total_attachments",
    "earliest_comment",
    "latest_comment"
  ],
  header: {
    country: "Country",
    comment_count: "Comments",
    document_count: "Documents",
    comments_with_attachments: "W/ Attachments",
    attachment_rate: "Attach Rate %",
    total_attachments: "Total Attachments",
    earliest_comment: "First Comment",
    latest_comment: "Last Comment"
  },
  format: {
    comment_count: d => d.toLocaleString(),
    document_count: d => d.toLocaleString(),
    comments_with_attachments: d => d.toLocaleString(),
    attachment_rate: d => `${d.toFixed(1)}%`,
    total_attachments: d => d.toLocaleString(),
    earliest_comment: d => d ? new Date(d).toLocaleDateString() : "N/A",
    latest_comment: d => d ? new Date(d).toLocaleDateString() : "N/A"
  },
  width: {
    country: 150,
    comment_count: 80,
    document_count: 80,
    comments_with_attachments: 90,
    attachment_rate: 80,
    total_attachments: 100,
    earliest_comment: 100,
    latest_comment: 100
  }
}));
```

  <!-- comment_text text // from 'comment' in API response
  comment_tracking_nbr varchar // from 'trackingNbr' in API response
  comment_category varchar // from 'category' in API response
  comment_subtype varchar // from 'subtype' in API response
  duplicate_comments int // from 'duplicateComments' in API response
  address1 varchar // from 'address1' in API response
  address2 varchar // from 'address2' in API response
  city varchar // from 'city' in API response
  state_province_region varchar // from 'stateProvinceRegion' in API response
  zip varchar // from 'zip' in API response
  country varchar // from 'country' in API response
  email varchar // from 'email' in API response
  first_name varchar // from 'firstName' in API response
  last_name varchar // from 'lastName' in API response
  phone varchar // from 'phone' in API response
  gov_agency varchar // from 'govAgency' in API response
  gov_agency_type varchar // from 'govAgencyType' in API response
  comment_postmark_date datetime // from 'postmarkDate' in API response
  comment_receive_date datetime // from 'receiveDate' in API response
  comment_attachment_count int // derived as the length of the 'included' array in the API response with 'type' = 'attachments'
  comment_details_download_date datetime [note: 'System date when the comment details were downloaded'] -->


```js
```

```js
// Display 5 random comment samples with text focus
const randomSample = filteredComments.length > 0 ? 
  d3.shuffle(filteredComments.slice()).slice(0, 5) : [];

display(html`<h4>Sample Comments ${searchTerm ? `(5 random from ${filteredComments.length} filtered records)` : `(5 random from ${comments.length} total)`}</h4>`);

// Function to decode HTML entities and escaped characters
function decodeHtmlEntities(text) {
  if (!text) return text;
  
  // Create a temporary DOM element to decode HTML entities
  const textarea = document.createElement('textarea');
  textarea.innerHTML = text;
  let decoded = textarea.value;
  
  // Handle additional common entities that might not be covered
  const entityMap = {
    '&ldquo;': '"',
    '&rdquo;': '"', 
    '&lsquo;': "'",
    '&rsquo;': "'",
    '&ndash;': '–',
    '&mdash;': '—',
    '&hellip;': '…',
    '&nbsp;': ' ',
    '&amp;': '&',
    '&lt;': '<',
    '&gt;': '>',
    '&quot;': '"',
    '&#39;': "'",
    '&#8220;': '"',
    '&#8221;': '"',
    '&#8216;': "'",
    '&#8217;': "'",
    '&#8211;': '–',
    '&#8212;': '—',
    '&#8230;': '…'
  };
  
  // Replace entities
  Object.entries(entityMap).forEach(([entity, replacement]) => {
    decoded = decoded.replace(new RegExp(entity, 'g'), replacement);
  });
  
  return decoded;
}

// Display sample comments with focus on text content
randomSample.forEach((comment, index) => {
  const contentSources = comment.comment_text_sources || [];
  
  display(html`
    <div style="border: 1px solid #ddd; margin: 10px 0; padding: 15px; border-radius: 5px;">
      <h5 style="margin-top: 0; color: #2563eb;">Sample Comment ${index + 1}</h5>
      <p><strong>Document:</strong> ${comment.document_title || 'Unknown'}</p>
      <p><strong>Country:</strong> ${comment.country || 'Unknown'} | 
         <strong>Agency:</strong> ${comment.gov_agency || 'N/A'} | 
         <strong>Attachments:</strong> ${comment.attachment_count || 0}</p>
      
      <div style="margin-top: 10px;">
        <strong>Comment Content:</strong>
        ${contentSources.length > 0 ? 
          contentSources.map((source, i) => html`
            <div style="margin: 8px 0; padding: 8px; border-radius: 3px; border-left: 3px solid #2563eb;">
              <div style="font-size: 0.75em; color: #666; margin-bottom: 5px;">
                <strong>Source ${i + 1}:</strong> ${source.source || 'Unknown'}
              </div>
              <div style="font-size: 0.85em; line-height: 1.4; word-wrap: break-word; white-space: pre-wrap;">
                ${source.text ? source.text.replace(/<[^>]*>/g, ' ') : 'No text available'}
              </div>
            </div>
          `) :
          html`<div style="color: #666; font-style: italic;">No content sources available</div>`
        }
      </div>
    </div>
  `);
  
  // Display the full JSON record
  display(comment);
});
```

```js
display(html`<h4>All Comments Data ${searchTerm ? `(filtered: ${filteredComments.length} of ${comments.length})` : `(${comments.length} total)`}</h4>`);
display(filteredComments);
```
