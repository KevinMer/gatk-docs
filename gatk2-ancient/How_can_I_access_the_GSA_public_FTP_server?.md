## How can I access the GSA public FTP server?

By Geraldine_VdAuwera

<p><strong>NOTE: This article will be deprecated in the near future as this information will be consolidated elsewhere.</strong></p>

<p>We make various files available for public download from the GSA FTP server, such as the GATK resource bundle and presentation slides. We also maintain a public upload feature for processing bug reports from users.</p>

<p>There are two logins to choose from depending on whether you want to upload or download something:</p>

<h3>Downloading</h3>

<pre class="code codeBlock" spellcheck="false">location: ftp.broadinstitute.org
username: gsapubftp-anonymous
password: &lt;blank&gt;
</pre>

<h3>Uploading</h3>

<pre class="code codeBlock" spellcheck="false">location: ftp.broadinstitute.org
username: gsapubftp
password: 5WvQWSfi
</pre>

<h3>Using a browser as FTP client</h3>

<p>If you use your browser as FTP client, make sure to include the login information in the address, otherwise you will access the general Broad Institute FTP instead of our team FTP. This should work as a direct link (for downloading only):</p>

<p><a rel="nofollow" href="denied:ftp://gsapubftp-anonymous@ftp.broadinstitute.org/bundle">ftp://gsapubftp-anonymous@ftp.broadinstitute.org/bundle</a></p>
