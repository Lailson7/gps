gps 
===

gps
<?php 
if($_GET['sources'])
    show_source(__FILE__);
else
    header('Content-Type: application/vnd.google-earth.kml+xml');
echo '<?xml version="1.0" encoding="UTF-8"?>'."\n";

$cnx = mysql_connect('localhost', 'user1', 'pass1');
mysql_select_db('tracker', $cnx);

if($_GET['entries'] <> ""){
  $entries = $_GET['entries'];
  }
elseif ($_POST['entries'] <> "") {
  $entries = $_POST['entries'];
  }  
else {
  $entries = 50;
  }
// echo $entries

if($_GET['imei'] <> "")
{
	$imei = $_GET['imei'];
}
elseif ($_POST['imei'] <> "") 
{
	$imei = $_POST['imei'];
} else {
	$imei = 0;
}

$step = 255 / $entries;
//echo $step;
$loop = 0;
$res1 = mysql_query("SELECT b.name as nomeBem, b.identificacao FROM bem b WHERE b.imei = $imei LIMIT 1");
while($data1 = mysql_fetch_assoc($res1))
{
	$nomeBem = $data1['nomeBem'];
	$identificacao = $data1['identificacao'];
}

$res = mysql_query("SELECT g.* FROM gprmc g WHERE g.gpsSignalIndicator = 'F' and g.imei = $imei ORDER BY g.id DESC LIMIT $entries");
$line_coordinates = "";
$ballons = "";
while($data = mysql_fetch_assoc($res))
{
    $trackerdate = ereg_replace("^(..)(..)(..)(..)(..)$","\\3/\\2/\\1 \\4:\\5",$data['date']);
    strlen($data['latitudeDecimalDegrees']) == 9 && $data['latitudeDecimalDegrees'] = '0'.$data['latitudeDecimalDegrees'];
    $g = substr($data['latitudeDecimalDegrees'],0,3);
    $d = substr($data['latitudeDecimalDegrees'],3);
    $latitudeDecimalDegrees = $g + ($d/60);
    $data['latitudeHemisphere'] == "S" && $latitudeDecimalDegrees = $latitudeDecimalDegrees * -1;


    strlen($data['longitudeDecimalDegrees']) == 9 && $data['longitudeDecimalDegrees'] = '0'.$data['longitudeDecimalDegrees'];
    $g = substr($data['longitudeDecimalDegrees'],0,3);
    $d = substr($data['longitudeDecimalDegrees'],3);
    $longitudeDecimalDegrees = $g + ($d/60);
    $data['longitudeHemisphere'] == "S" && $longitudeDecimalDegrees = $longitudeDecimalDegrees * -1;

    $longitudeDecimalDegrees = $longitudeDecimalDegrees * -1;

    $speed = $data['speed'] * 1.609;
	
	$line_coordinates .= "$longitudeDecimalDegrees,$latitudeDecimalDegrees,0\n";
	$line_coordinates_green = "$longitudeDecimalDegrees,$latitudeDecimalDegrees,0\n";

	if ($loop != 0) {
		$ballons .= '
		<Placemark>
			<name>'.$nomeBem.' - '.$identificacao.'</name>
			<styleUrl>#highlightPlacemark</styleUrl>
			<description>Velocidade : '.floor($speed).'Km/h - Data : '.date('d/m/Y H:i:s', strtotime($data['date'])).' &lt;br/&gt; Lat: '.$longitudeDecimalDegrees.', Long:'.$latitudeDecimalDegrees. '</description>
			<Point>
			  <coordinates>'."$longitudeDecimalDegrees,$latitudeDecimalDegrees,0".'</coordinates>
			</Point>
		</Placemark>
		';
	} else {
		//O ultimo registro obtido pelo gps fica verde; o ultimo é o primeiro da lista. ORDER BY DESC.
		if ($loop == 0) {
			$greenBallons = '
			<Placemark>
				<name>'.$nomeBem.' - '.$identificacao.'</name>
				<styleUrl>#highlightPlacemarkGreen</styleUrl>
				<description>Velocidade : '.floor($speed).'Km/h - Data : '.date('d/m/Y H:i:s', strtotime($data['date'])).' &lt;br/&gt; Lat: '.$longitudeDecimalDegrees.', Long:'.$latitudeDecimalDegrees. '</description>
				<Point>
				  <coordinates>'."$longitudeDecimalDegrees,$latitudeDecimalDegrees,0".'</coordinates>
				</Point>
			</Placemark>
		';
		}
	}
	
	$loop++;
    
}
mysql_close($cnx);
?>
<kml xmlns="http://earth.google.com/kml/2.1">
  <Document>
    <name>Tracker Map</name>
    <description>Tracker</description>

    <Style id="highlightPlacemark">
      <IconStyle>
        <Icon>
          <href>http://google-maps-icons.googlecode.com/files/suv.png</href>
        </Icon>
      </IconStyle>
    </Style>

    <Style id="highlightPlacemarkGreen">
      <IconStyle>
        <Icon>
          <href>http://google-maps-icons.googlecode.com/files/amphitheater-tourism.png</href>
        </Icon>
      </IconStyle>
    </Style>

    <Style id="redLine">
      <LineStyle>
        <color>ff0000ff</color>
        <width>4</width>
      </LineStyle>
    </Style>

    <Style id="BalloonStyle">
      <BalloonStyle>
        <!-- a background color for the balloon -->
        <bgColor>ffffffbb</bgColor>
        <!-- styling of the balloon text -->
        <text><![CDATA[
        <b><font color="#CC0000" size="+3">$[name]</font></b>
        <br/><br/>
        <font face="Courier">$[description]</font>
        <br/><br/>
        Extra text that will appear in the description balloon
        <br/><br/>
        <!-- insert the to/from hyperlinks -->
        $[geDirections]
        ]]></text>
      </BalloonStyle>
    </Style>

    <Style id="greenPoint">
      <LineStyle>
        <color>ff009900</color>
        <width>4</width>
      </LineStyle>
    </Style>

    <Placemark>
      <name>Red Line</name>
      <styleUrl>#redLine</styleUrl>
      <LineString>
        <altitudeMode>relative</altitudeMode>
        <coordinates>
			<?php echo $line_coordinates; ?>
        </coordinates>
      </LineString>
    </Placemark>
	
	<?php echo $greenBallons; ?>

    <?php echo $ballons; ?>

  </Document>
</kml>
