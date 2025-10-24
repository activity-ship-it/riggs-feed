#!/usr/bin/env python3
import sys, os, re, datetime
from xml.etree import ElementTree as ET

# Usage:
#   python scripts/add_item.py "<title>" "<link>" "<description>" [guid]
#
# Example:
#   python scripts/add_item.py "Trail Report – Titus Canyon" "https://4x4trailrunners.com/trails/titus-canyon" "Fresh trail conditions & tips"

def rfc2822_now():
    # Publer/GitHub Pages are fine with RFC 2822 timestamps
    return datetime.datetime.utcnow().strftime("%a, %d %b %Y %H:%M:%S +0000")

def ensure_channel(tree):
    rss = tree.getroot()
    if rss.tag != "rss":
        raise RuntimeError("Root element must be <rss>")
    ch = rss.find("channel")
    if ch is None:
        ch = ET.SubElement(rss, "channel")
    return ch

def load_tree(path):
    if not os.path.exists(path):
        # create minimal feed skeleton
        rss = ET.Element("rss", {"version": "2.0"})
        ch = ET.SubElement(rss, "channel")
        ET.SubElement(ch, "title").text = "Riggs Autoposts"
        ET.SubElement(ch, "link").text = "https://4x4trailrunners.com/"
        ET.SubElement(ch, "description").text = "Automated feed from Riggs"
        ET.SubElement(ch, "lastBuildDate").text = rfc2822_now()
        return ET.ElementTree(rss)
    return ET.parse(path)

def insert_item(channel, title, link, desc, guid=None, pubdate=None, max_items=50):
    if pubdate is None:
        pubdate = rfc2822_now()

    # Use a stable GUID: prefer explicit guid, otherwise fall back to the link
    if guid is None or not guid.strip():
        guid = link

    # De-dup: if an item with this GUID already exists, skip
    existing_guids = {i.findtext("guid") for i in channel.findall("item")}
    if guid in existing_guids:
        # Optional: you could update the existing item's description/pubDate here instead
        return

    # Build <item>
    item = ET.Element("item")
    ET.SubElement(item, "title").text = title
    ET.SubElement(item, "link").text = link
    ET.SubElement(item, "description").text = desc
    ET.SubElement(item, "pubDate").text = pubdate
    ET.SubElement(item, "guid").text = guid

    # Insert newest first
    items = channel.findall("item")
    if items:
        channel.insert(list(channel).index(items[0]), item)
    else:
        channel.append(item)

    # Update lastBuildDate
    lbd = channel.find("lastBuildDate")
    if lbd is None:
        lbd = ET.SubElement(channel, "lastBuildDate")
    lbd.text = rfc2822_now()

    # Trim old items
    items = channel.findall("item")
    for old in items[max_items:]:
        channel.remove(old)


    # update lastBuildDate
    lbd = channel.find("lastBuildDate")
    if lbd is None:
        lbd = ET.SubElement(channel, "lastBuildDate")
    lbd.text = rfc2822_now()

    # trim old items
    items = channel.findall("item")
    for old in items[max_items:]:
        channel.remove(old)

def main():
    if len(sys.argv) < 4:
        print("Usage: python scripts/add_item.py \"<title>\" \"<link>\" \"<description>\" [guid]", file=sys.stderr)
        sys.exit(1)

    title, link, desc = sys.argv[1], sys.argv[2], sys.argv[3]
    guid = sys.argv[4] if len(sys.argv) > 4 else None

    feed_path = os.path.join(os.path.dirname(__file__), "..", "feed.xml")
    feed_path = os.path.abspath(feed_path)

    tree = load_tree(feed_path)
    channel = ensure_channel(tree)
    insert_item(channel, title, link, desc, guid=guid)

    # Pretty print (ElementTree doesn't indent by default)
    ET.indent(tree, space="  ", level=0)  # Python 3.9+
    with open(feed_path, "wb") as f:
        tree.write(f, encoding="UTF-8", xml_declaration=True)

    print(f"Updated {feed_path}")

if __name__ == "__main__":
    main()
