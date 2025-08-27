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
const lastModified = comments && comments.length > 0
  ? new Date(Math.max(...comments
      .map(d => d.comment_details_download_date)
      .filter(d => d)
      .map(d => new Date(d))))
  : null;
const href = commentsFile.href;
const downloadName = "public_comments_" + lastModified.toISOString().replace(/[:.]/g, "-") + ".json";
```

<div style="margin-bottom: 1rem;">
  <small style="color: #666; font-size: 0.75em;">
    Data as of ${lastModified.toLocaleString()} | 
    <a href="./data/public_comments.json" download="public_comments.json" style="color: #0066cc;">Download raw unfiltered data</a>
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

```js
// Interactive filtering state management
const filterState = {
  selectedDocuments: new Set(),
  selectedAgencies: new Set(),
  selectedAgencyTypes: new Set(),
  selectedCountries: new Set()
};

// Function to update filters
function updateFilter(type, value) {
  const set = filterState[type];
  if (set.has(value)) {
    set.delete(value);
  } else {
    set.add(value);
  }
  // Trigger re-computation by updating a reactive cell
  filterTrigger.value = Date.now();
}

// Create a reactive trigger for filter updates
const filterTrigger = Inputs.input(0);
const filterUpdate = Generators.input(filterTrigger);

// Function to get active filters display
function getActiveFiltersDisplay() {
  const filters = [];
  if (filterState.selectedDocuments.size > 0) {
    filters.push(`Documents: ${Array.from(filterState.selectedDocuments).join(', ')}`);
  }
  if (filterState.selectedAgencies.size > 0) {
    filters.push(`Agencies: ${Array.from(filterState.selectedAgencies).join(', ')}`);
  }
  if (filterState.selectedAgencyTypes.size > 0) {
    filters.push(`Agency Types: ${Array.from(filterState.selectedAgencyTypes).join(', ')}`);
  }
  if (filterState.selectedCountries.size > 0) {
    filters.push(`Countries: ${Array.from(filterState.selectedCountries).join(', ')}`);
  }
  return filters.length > 0 ? filters.join(' | ') : 'No filters applied';
}

// Function to clear all filters
function clearAllFilters() {
  filterState.selectedDocuments.clear();
  filterState.selectedAgencies.clear();
  filterState.selectedAgencyTypes.clear();
  filterState.selectedCountries.clear();
  filterTrigger.value = Date.now();
}
```

```js
// Interactive filtering state management
const filterState = {
  selectedDocuments: new Set(),
  selectedAgencies: new Set(),
  selectedAgencyTypes: new Set(),
  selectedCountries: new Set()
};

// Function to update filters
function updateFilter(type, value) {
  const set = filterState[type];
  if (set.has(value)) {
    set.delete(value);
  } else {
    set.add(value);
  }
  // Trigger re-computation by updating a reactive cell
  filterTrigger.value = Date.now();
}

// Create a reactive trigger for filter updates
const filterTrigger = Inputs.input(0);
const filterUpdate = Generators.input(filterTrigger);

// Function to get active filters display
function getActiveFiltersDisplay() {
  const filters = [];
  if (filterState.selectedDocuments.size > 0) {
    filters.push(`Documents: ${Array.from(filterState.selectedDocuments).join(', ')}`);
  }
  if (filterState.selectedAgencies.size > 0) {
    filters.push(`Agencies: ${Array.from(filterState.selectedAgencies).join(', ')}`);
  }
  if (filterState.selectedAgencyTypes.size > 0) {
    filters.push(`Agency Types: ${Array.from(filterState.selectedAgencyTypes).join(', ')}`);
  }
  if (filterState.selectedCountries.size > 0) {
    filters.push(`Countries: ${Array.from(filterState.selectedCountries).join(', ')}`);
  }
  return filters.length > 0 ? filters.join(' | ') : 'No filters applied';
}

// Function to clear all filters
function clearAllFilters() {
  filterState.selectedDocuments.clear();
  filterState.selectedAgencies.clear();
  filterState.selectedAgencyTypes.clear();
  filterState.selectedCountries.clear();
  filterTrigger.value = Date.now();
}
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
// Function to download filtered data
function downloadFilteredData() {
  const dataToDownload = filteredComments;
  const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
  const filename = `filtered_comments_${timestamp}.json`;
  
  const dataStr = JSON.stringify(dataToDownload, null, 2);
  const dataBlob = new Blob([dataStr], {type: 'application/json'});
  const url = URL.createObjectURL(dataBlob);
  
  const link = document.createElement('a');
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
}

// Create download button
const downloadButton = Inputs.button("Download filtered data", {
  reduce: () => downloadFilteredData()
});
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

// Function to highlight search terms in text
function highlightSearchTerms(text, searchTerm) {
  if (!text || !searchTerm) return text;
  
  // Escape special regex characters in search term
  const escapedTerm = searchTerm.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  
  // Create regex for case-insensitive matching
  const regex = new RegExp(`(${escapedTerm})`, 'gi');
  
  // Split text by search term but keep the delimiters
  const parts = text.split(regex);
  
  // Return HTML template with highlighted parts
  return html`${parts.map((part, i) => 
    regex.test(part) && part.toLowerCase() === searchTerm.toLowerCase() 
      ? html`<mark style="background-color: #ffeb3b; padding: 1px 2px; border-radius: 2px;">${part}</mark>`
      : part
  )}`;
}

// Display sample comments with focus on text content
randomSample.forEach((comment, index) => {
  const contentSources = comment.comment_text_sources || [];
  
  display(html`
    <div style="border: 1px solid #ddd; margin: 10px 0; padding: 15px; border-radius: 5px;">
      <h5 style="margin-top: 0; color: #2563eb;">Sample Comment ${index + 1}</h5>
      <p><strong>Document:</strong> ${highlightSearchTerms(comment.document_title || 'Unknown', searchTerm)}</p>
      <p><strong>Country:</strong> ${highlightSearchTerms(comment.country || 'Unknown', searchTerm)} | 
         <strong>Agency:</strong> ${highlightSearchTerms(comment.gov_agency || 'N/A', searchTerm)} | 
         <strong>Attachments:</strong> ${comment.attachment_count || 0}</p>
      
      <div style="margin-top: 10px;">
        <strong>Comment Content:</strong>
        ${contentSources.length > 0 ? 
          contentSources.map((source, i) => html`
            <div style="margin: 8px 0; padding: 8px; border-radius: 3px; border-left: 3px solid #2563eb;">
              <div style="font-size: 0.75em; color: #666; margin-bottom: 5px;">
                <strong>Source ${i + 1}:</strong> ${(() => {
                  const sourceText = source.source || 'Unknown';
                  // Check if source appears to be a URL (starts with http)
                  if (sourceText.startsWith('http')) {
                    return html`<a href="${sourceText}" target="_blank" rel="noopener noreferrer" style="color: #0066cc; text-decoration: underline;">${sourceText}</a>`;
                  } else {
                    return sourceText;
                  }
                })()}
              </div>
              <div style="font-size: 0.85em; line-height: 1.4; word-wrap: break-word; white-space: pre-wrap;">
                ${source.text ? highlightSearchTerms(decodeHtmlEntities(source.text.replace(/<[^>]*>/g, ' ')), searchTerm) : 'No text available'}
              </div>
            </div>
          `) :
          html`<div style="color: #666; font-style: italic;">No content sources available</div>`
        }
      </div>
    </div>
  `);
});
```
---

```js
display(html`<h4>Comments Data ${searchTerm ? `(filtered: ${filteredComments.length} of ${comments.length})` : `(${comments.length} total)`}</h4>`);
display(filteredComments);
```
<div>
    ${downloadButton}
</div>
