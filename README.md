
Download attachments from an IMAP mailbox.

Usage examples:
  # Prompt for password:
  python download_attachments.py --email you@example.com

  # Provide password via environment variable (safer than hardcoding):
  EMAIL_PASSWORD=your_app_password python download_attachments.py --email you@example.com --search '(UNSEEN)'

Notes:
- For Gmail with 2FA you must create an App Password (or use OAuth2); do NOT use your main account password.
- Do not hardcode credentials into scripts.
"""
import imaplib
import email
import os
import re
import argparse
import getpass
import logging
from typing import Optional

logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")


def sanitize_filename(name: str) -> str:
    name = os.path.basename(name)
    # Replace any char that's not alphanumeric, dot, underscore, or hyphen
    return re.sub(r"[^A-Za-z0-9._-]", "_", name)


def unique_path(directory: str, filename: str) -> str:
    base, ext = os.path.splitext(filename)
    candidate = filename
    i = 1
    while os.path.exists(os.path.join(directory, candidate)):
        candidate = f"{base}_{i}{ext}"
        i += 1
    return os.path.join(directory, candidate)


def download_attachments(
    email_account: str,
    password: str,
    imap_server: str = "imap.gmail.com",
    mailbox: str = "INBOX",
    search_criterion: str = "ALL",
    save_dir: str = "attachments",
) -> None:
    if not os.path.exists(save_dir):
        os.makedirs(save_dir, exist_ok=True)

    mail = None
    try:
        logging.info("Connecting to IMAP server %s ...", imap_server)
        mail = imaplib.IMAP4_SSL(imap_server)
        mail.login(email_account, password)
        mail.select(mailbox)

        logging.info("Searching mailbox %s with criterion: %s", mailbox, search_criterion)
        typ, data = mail.search(None, search_criterion)
        if typ != "OK":
            logging.error("Search failed: %s", typ)
            return

        mail_ids = data[0].split()
        logging.info("Found %d messages", len(mail_ids))

        downloaded = 0
        for num in mail_ids:
            typ, msg_data = mail.fetch(num, "(RFC822)")
            if typ != "OK":
                logging.warning("Failed to fetch message %s: %s", num.decode() if isinstance(num, bytes) else num, typ)
                continue

            # msg_data can be a list like [(b'1 (RFC822 {....}', b'raw bytes'), b')'] - find the bytes part
            raw_bytes = None
            for part in msg_data:
                if isinstance(part, tuple):
                    raw_bytes = part[1]
                    break
            if raw_bytes is None:
                logging.warning("No message bytes for %s", num)
                continue

            msg = email.message_from_bytes(raw_bytes)

            for part in msg.walk():
                # skip container parts
                if part.get_content_maintype() == "multipart":
                    continue
                # Some attachments may not have Content-Disposition but still have filename
                filename = part.get_filename()
                if not filename:
                    # Try to get a filename from content-type header params
                    cd = part.get("Content-Disposition", "")
                    if "filename=" in cd:
                        filename = cd.split("filename=")[-1].strip(' "')

                if not filename:
                    continue

                filename = sanitize_filename(filename)
                filepath = unique_path(save_dir, filename)
                payload = part.get_payload(decode=True)
                if payload is None:
                    logging.warning("Attachment %s had no payload; skipping", filename)
                    continue

                with open(filepath, "wb") as f:
                    f.write(payload)
                logging.info("Downloaded: %s", filepath)
                downloaded += 1

        logging.info("Done. Total attachments downloaded: %d", downloaded)

    except imaplib.IMAP4.error as e:
        logging.error("IMAP error: %s", e)
    except Exception as e:
        logging.exception("Unexpected error:")
    finally:
        if mail is not None:
            try:
                mail.logout()
            except Exception:
                pass


def main():
    parser = argparse.ArgumentParser(description="Download attachments from an IMAP mailbox")
    parser.add_argument("--email", required=True, help="Email address / account username")
    parser.add_argument(
        "--password",
        help="Password or app password. If omitted, will try $EMAIL_PASSWORD then prompt.",
    )
    parser.add_argument("--server", default="imap.gmail.com", help="IMAP server (default: imap.gmail.com)")
    parser.add_argument("--mailbox", default="INBOX", help="Mailbox to search (default: INBOX)")
    parser.add_argument("--search", default="ALL", help='IMAP SEARCH criterion (default: "ALL"). Examples: "UNSEEN", \'(UNSEEN FROM "name@example.com")\'')
    parser.add_argument("--dir", default="attachments", help="Directory to save attachments (default: attachments)")
    args = parser.parse_args()

    password = args.password or os.environ.get("EMAIL_PASSWORD")
    if not password:
        password = getpass.getpass(prompt=f"Password (or app password) for {args.email}: ")

    download_attachments(
        email_account=args.email,
        password=password,
        imap_server=args.server,
        mailbox=args.mailbox,
        search_criterion=args.search,
        save_dir=args.dir,
    )


if __name__ == "__main__":
    main()