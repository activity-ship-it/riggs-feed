#!/usr/bin/env python3
import sys, os, datetime
from xml.etree import ElementTree as ET
import urllib.parse as _url

# Usage:
#   python scripts/add_item.py "<title>" "<link>" "<description>" [guid]
#
# Example:
#   python scripts/add_item.py "Trail Report – Titus Canyon" \
#     "https://4x4trailrunners.com/trails/titus-canyon?utm_source=x" \
#     "Fresh trail conditions & tips"

TRACKING_PARAMS = {
    "utm_source","utm_medium","utm_campaign","utm_term","utm_content","utm_id",
    "fbclid","gclid","igshid","mc_cid","mc_eid"
}

def rfc2822_now():
    # RFC 2822 timestamp in UTC
    return datetime.datetime.utcnow().strftime("%a, %d %b %Y %H:%M:%S +0000")

def _canonicalize_link(link: str) -> str:
    """Strip common tracking params so the same article URL maps to one GUID."""
    if not link:
        return link
    try:
        p = _url.urlsplit(link)
        q = _url.parse_qsl(p.query, keep_blank_values=True)
        q = [(k, v) for (k, v) in q if k not in TRACKING_PARAMS]
        new_query = _url.urlencode(q, doseq=True)
        return _url.urlunsplit((p.scheme, p.netloc, p.path, new_query, p.fragment))
    except Exception:
        return link  # fail open

def ensure_channel(tree):
    rss = tree.getroot()
    if rss.tag != "rss":
        raise RuntimeError("Root element must be <rss>")
    ch = rss.find("channel")
    if ch is None:
        ch = ET.SubElement(rss, "channel")
    # Ensure minimal metadata exists
    if ch.find("title") is None:
        ET.SubElement(ch, "title").text = "Riggs Autoposts"
    if ch.find("link") is None:
        ET.SubElement(ch, "link").text = "https://4x4trailrunners.com/"
    if ch.find("description") is None:
        ET.SubElement(ch, "description").text = "Automated feed from Riggs"
    if ch.find("lastBuildDate") is None:
        ET.SubElement(ch, "lastBuildDate").text = rfc2822_now()
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

    link_canonical = _canonicalize_link(link or "")
    # Stable GUID: explicit guid > canonical link > title
    stable_guid = (guid or "").strip() or link_canonical or (title or "").strip()

    # De-dup: skip if GUID already exists (OPTION: update instead of skip)
    existing_items = channel.findall("item")
    existing_by_guid = {i.findtext("guid"): i for i in existing_items}
    if stable_guid in existing_by_guid:
        # OPTIONAL update-in-place:
        # existing = existing_by_guid[stable_guid]
        # if desc: existing.find("description").text = desc
        # existing.find("pubDate").text = pubdate
        return

    # Build <item>
    item = ET.Element("item")
    ET.SubElement(item, "title").text = title
    ET.SubElement(item, "link").text = link_canonical or link
    ET.SubElement(item, "description").text = desc
    ET.SubElement(item, "pubDate").text = pubdate

    guid_el = ET.SubElement(item, "guid")
    guid_el.text = stable_guid
    if stable_guid.startswith("http://") or stable_guid.startswith("https://"):
        guid_el.set("isPermaLink", "true")
    else:
        guid_el.set("isPermaLink", "false")

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

def main():
    if len(sys.argv) < 4:
        print('Usage: python scripts/add_item.py "<title>" "<link>" "<description>" [guid]', file=sys.stderr)
        sys.exit(1)

    title, link, desc = sys.argv[1], sys.argv[2], sys.argv[3]
    guid = sys.argv[4] if len(sys.argv) > 4 else None

    feed_path = os.path.join(os.path.dirname(__file__), "..", "feed.xml")
    feed_path = os.path.abspath(feed_path)

    tree = load_tree(feed_path)
    channel = ensure_channel(tree)
    insert_item(channel, title, link, desc, guid=guid)

    # Pretty print (Python 3.9+)
    ET.indent(tree, space="  ", level=0)
    with open(feed_path, "wb") as f:
        tree.write(f, encoding="UTF-8", xml_declaration=True)

    print(f"Updated {feed_path}")

if __name__ == "__main__":
    main()
