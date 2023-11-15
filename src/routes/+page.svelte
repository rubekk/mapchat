<script>
    import { onMount } from "svelte";
    import { initializeApp } from "firebase/app";
	import { getDatabase, ref, set, push, onValue, onDisconnect, child } from "firebase/database";
    import { firebaseConfig } from "$lib/firebaseConfig";
    import "./../app.css";

    const firebaseApp=initializeApp(firebaseConfig);
    const db=getDatabase();

    let leaflet,
        map,
        markerGroup,
        myLocation,
        showMyLocation=false,
        showWhatInfo=false,
        mapLoaded=false,
        increasedUsers=false,
        showMsg=true,
        showError=false,
        errorMsg="",
        userId="",
        activeUsers=0,
        userCoords=[],
        activeUsersCoords=[],
        defaultViewCoords=['27.665', '85.331'],
        markers=[],
        chatData=[],
        days=[],
        msg="",
        msgDiv;
    
    // get user location
    const getLocation=()=>{
        if (navigator.geolocation) {
            navigator.geolocation.getCurrentPosition(
                position=> userCoords= [ position.coords.latitude.toFixed(3), position.coords.longitude.toFixed(3) ],
                ()=> {
                    errorMsg="Sorry! your location couldn't be tracked. Your location will be set to a random location. Please use different device / browser for exact location.";
                    showError=true;

                    userCoords=['27.555','28.555']

                    setTimeout(()=> showError=false, 7500);
                }
            );
        } 
        else {
            errorMsg="Sorry! your location couldn't be tracked. Your location will be set to a random location. Please use different device / browser for exact location.";
            showError=true;

            userCoords=['27.555','28.555']

            setTimeout(()=> showError=false, 7500);
        }
    }

    // send message
    const sendMessage=()=>{
        if(msg && userCoords.length>0){
            const chatListRef = ref(db, 'chat');
            const newChatRef = push(chatListRef);
            
            let currentDate=new Date();
            let data={
                msg: msg,
                time: `${String(currentDate.getHours()).length==1?'0'+String(currentDate.getHours()):currentDate.getHours()}:${String(currentDate.getMinutes()).length==1?'0'+String(currentDate.getMinutes()):currentDate.getMinutes()}:${String(currentDate.getSeconds()).length==1?'0'+String(currentDate.getSeconds()):currentDate.getSeconds()}`,
                date: `${currentDate.getFullYear()}/${currentDate.getMonth()+1}/${currentDate.getDate()}`,
                coords: userCoords
            };

            // data={
            //     msg: "this is a great message",
            //     time: "09:00:00",
            //     date: "15/10/2023",
            //     coords: [27.555, 28.555]
            // }
    
            set(newChatRef, data);

            msg="";
        }
        else if(userCoords.length==0) getLocation();
    }

    const increaseActiveUsers=()=>{  
        if(userCoords.length==0) getLocation();

        if(!increasedUsers){
            const activeRef = ref(db, 'active');
            const newActiveRef = push(activeRef);
            
            
            set(newActiveRef, {
                uid: Math.floor(Math.random()*500)
            })
            
            userId=newActiveRef.key;
            
            onDisconnect(ref(db,'active/' + userId)).set({
                uid: null
            })
        }
        
        increasedUsers=true;
    }

    // handle individual message click
    const handleMessageClick=index=>{
        let includes=false;
        let includeIndex=0;

        markers.forEach((elem,i)=>{
            if(elem.coords[0]==chatData[index].coords[0] && elem.coords[1]==chatData[index].coords[1]) {
                includes=true;
                includeIndex=i;
            }
        })

        if(!includes) markers.push(chatData[index])
        else markers[includeIndex]=chatData[index];

        showMarkers(chatData[index].coords, includeIndex);
    }

    const handleInputClick=()=>{
        if(userCoords.length==0) getLocation();
    }

    // show markers on message load
    const showMarkersOnMessageLoad=()=>{
        days=[];
        
        chatData.forEach(elem=>{
            let includes=false;
            let includeIndex=0;

            markers.forEach((item,i)=>{
                if(elem.coords[0]==item.coords[0] && elem.coords[1]==item.coords[1]) {
                    includes=true;
                    includeIndex=i;
                }
            })

            if(!includes) markers.push(elem);
            else markers[includeIndex]=elem;
        })

        showMarkers();
    }

    // show markers
    const showMarkers=(viewCoords=defaultViewCoords, index=-1)=>{
        map.removeLayer(markerGroup);
        markerGroup=leaflet.layerGroup().addTo(map);

        if(index>=markers.length) index=markers.length-1;
        else if(index==-1 && markers.length>0 && chatData.length>0){
            markers.forEach((elem,i)=>{
                if(elem.coords[0]==chatData[chatData.length-1].coords[0] && elem.coords[1]==chatData[chatData.length-1].coords[1])  index=i;
            })
        }

        markers.forEach((elem,i)=>{
            if(index==i){
                
                leaflet.marker(elem.coords).addTo(markerGroup).bindPopup(leaflet.popup().setContent(
                    `<div>
                        <p style="font-style: italic; font-size: .8rem"> ${elem.msg} </p>
                        <p style="margin: -17px 0; float: right; font-style: italic; font-size: .6rem; color: #797979;"> ${elem.time} </p>
                    </div>`
                )).openPopup();
            }
            else{
                leaflet.marker(elem.coords).addTo(markerGroup).bindPopup(leaflet.popup().setContent(
                    `<div>
                        <p style="font-style: italic; font-size: .8rem"> ${elem.msg} </p>
                        <p style="margin: -17px 0; float: right; font-style: italic; font-size: .6rem; color: #797979;"> ${elem.time} </p>
                    </div>`
                ));
            }
        })

        map.setView(viewCoords, 12);
    }

    // show me
    const showMe=()=>{
        if(userCoords.length==0) getLocation();
        else {
            showMyLocation=!showMyLocation;
            map.removeLayer(myLocation);

            if(!showMyLocation) return;
            
            myLocation=leaflet.layerGroup().addTo(map);
            leaflet.marker(userCoords).addTo(myLocation);
            map.setView(userCoords, 12);
        }
    }

    // handle message dates
    const handleMessageDate=date=>{
        let day=date[0]+date[1];

        if(days.includes(day)) return false;
        else{
            days.push(day);
            return true;
        }
    }

    // handle show message
    const handleShowMsg=()=>{
        days=[];

        showMsg=!showMsg;

        if(showMsg && msgDiv) msgDiv.scrollTop = msgDiv.scrollHeight;
    }

    // handle show what info
    const handleWhatInfo=()=>{
        showWhatInfo=!showWhatInfo;
    }

    // get day of week
    const getDayOfWeek=date=>{
        const dayOfWeek = new Date(date).getDay();    
        return isNaN(dayOfWeek) ? null : 
            ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'][dayOfWeek];
    }

    // keep upto date with all chat messages from firebase
    onValue(ref(db, "chat"), (snapshot) => {
        if(snapshot.val()){
            chatData = Object.values(snapshot.val());

            defaultViewCoords=chatData[chatData.length-1].coords;
        }
    });

    // keep upto date with active users from firebase
    onValue(ref(db, "active"), (snapshot) => {
        if(snapshot.val()){
            activeUsers=Object.values(snapshot.val());

            activeUsersCoords=[];
            activeUsers.forEach(elem=>{
                activeUsersCoords.push(elem.uid);
            })
        }
    });

    // draw map first
    onMount(async ()=>{
        leaflet= await import("leaflet");

        map = leaflet.map('map', { zoomControl: false }).setView(defaultViewCoords, 12);

        leaflet.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

        leaflet.control.zoom({
            position: 'bottomright'
        }).addTo(map);

        markerGroup=leaflet.layerGroup().addTo(map);
        myLocation=leaflet.layerGroup().addTo(map);

        mapLoaded=true;
    })

    $: if(chatData && mapLoaded) showMarkersOnMessageLoad();

    $: if(userCoords.length>0) set(ref(db, 'active/' + userId), {
            uid: userCoords[0]+"*"+userCoords[1]
        });
</script>

<div class="container" on:click={increaseActiveUsers}>
    <div id="map"></div>

    <div class="title">
        <span class="title-map">Map</span><span class="title-chat">Chat</span>
        <br>
        <span class="sub-title">नक्शा मा कुरा</span>
    </div>
    
    <div on:click={showMe} class={showMyLocation? 'show-me anim': 'show-me'} title="locate me">
        <i class="fa-solid fa-location-dot"></i>
    </div>
    
    <div class="active">Active users: {activeUsers?activeUsers.length:0}</div>

    <div on:click={handleWhatInfo} class={showWhatInfo? 'what anim': 'what'} title="what">
        <i class="fa-solid fa-question"></i>
    </div>
    

    <div class="chat">
        <i on:click={handleShowMsg} class={`fa-solid ${showMsg?'fa-chevron-down':'fa-chevron-up'} cross-chat`} title={showMsg?'minimize':'expand'}></i>
        {#if chatData.length>0 && showMsg}
        <div bind:this={msgDiv} class="message">
            {#each chatData as data,index}
                {#if handleMessageDate(data.date)}
                    <p class="msg-date">{getDayOfWeek(data.date)} {data.date}</p>
                {/if}
                <div on:click={()=>handleMessageClick(index)} class={data.coords[0]==userCoords[0] && data.coords[1]==userCoords[1]? 'ind-message right-side':'ind-message'}>
                    <i class="fa-solid fa-chevron-right"></i>
                    <div class="msg">{data.msg}</div>
                    <div class="time">{data.time}</div>
                </div>
            {/each}
        </div>
        {/if}
        <div class="inp">
            <input bind:value={msg} on:click={handleInputClick} on:keydown={e=>{if(e.keyCode==13) sendMessage()}} type="text" placeholder="Enter a message">
            <button>
                <i on:click={sendMessage} class="fa-solid fa-paper-plane"></i>
            </button>
        </div>
    </div>

    {#if showWhatInfo}
    <div class="what-info">
        MapChat is a realtime location based chat website. Created by Rubek
    </div>
    {/if}

    {#if showError}
    <div class="error">
        {errorMsg}
    </div>
    {/if}
</div>